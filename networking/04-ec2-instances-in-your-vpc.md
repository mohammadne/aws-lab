# Module 04 — EC2 Instances in Your VPC

← [All tutorials](../README.md) · **Networking tutorial**, module 4 of 8

Time to put servers into the network. An **EC2 instance** is a virtual server. In this module you launch two of them and see exactly how they plug into the subnets, routes, and firewalls you built:

- `web-1`, a **public** test web server in `public-a`, reachable from your browser;
- `app-1`, a **private** application server in `app-a`, with no public IP at all.

---

## 1. Launch an instance, flag by flag

Here's the full command for `web-1`. Every option is explained below it.

```bash
AMI_ID=$(aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --query Parameter.Value --output text); save AMI_ID
aws ec2 create-key-pair --key-name lab-key --key-type ed25519 --query KeyMaterial --output text > ~/lab-key.pem
chmod 400 ~/lab-key.pem

cat > /tmp/web.sh <<'USERDATA'
#!/bin/bash
dnf install -y nginx
TOKEN=$(curl -sX PUT http://169.254.169.254/latest/api/token -H "X-aws-ec2-metadata-token-ttl-seconds: 300")
md() { curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/$1; }
echo "<h1>web-1 $(md local-ipv4) in $(md placement/availability-zone)</h1>" > /usr/share/nginx/html/index.html
systemctl enable --now nginx
USERDATA

WEB_ID=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t3.micro \
  --subnet-id $PUB_A \
  --security-group-ids $SG_WEB \
  --key-name lab-key \
  --metadata-options HttpTokens=required \
  --user-data file:///tmp/web.sh \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=web-1}]' \
  --query 'Instances[0].InstanceId' --output text); save WEB_ID
```

| Option | What it means |
|---|---|
| `--image-id` | The **AMI** (Amazon Machine Image): the disk image the server boots from (OS + preinstalled software). AMIs are regional. The `ssm get-parameter` line looks up the latest official Amazon Linux 2023 AMI for your Region, so you never hard-code an ID |
| `--instance-type` | The server size: CPU, memory, and **network bandwidth**. `t3.micro` = 2 vCPUs, 1 GiB. The name reads as family `t` (burstable), generation `3`, size `micro` |
| `--subnet-id` | **The most important networking choice.** It fixes the **AZ** and the **IP range** the server gets, and therefore which route table (internet or not) applies. You can't change it after launch |
| `--security-group-ids` | One or more security groups (Module 03). If omitted, the VPC's default security group is used |
| `--key-name` | An SSH key pair. AWS installs the public key on the server at first boot, and you keep the private key (`~/lab-key.pem`). It's optional: Section 4 shows ways in without keys |
| `--metadata-options HttpTokens=required` | Protects the **instance metadata service**, a local web endpoint (`169.254.169.254`) the server uses to learn about itself and to get IAM role credentials. "Required tokens" (IMDSv2) blocks a common class of attacks. Always set it |
| `--user-data` | A script run **once, at first boot**. Here it installs nginx and writes a page showing the server's private IP and AZ (read from the metadata service) |
| *(not used here)* `--iam-instance-profile` | Gives the server an **IAM role**, so software on it can call AWS APIs without access keys. See the [IAM tutorial, Module 02](../iam/02-how-services-use-iam.md) |

Because `public-a` has *auto-assign public IP* turned on (Module 02), `web-1` also gets a public IP. Now launch the private `app-1`. It runs a tiny HTTP service on port 8080 and gets no public IP, because `app-a` doesn't assign one:

```bash
cat > /tmp/app.sh <<'USERDATA'
#!/bin/bash
mkdir -p /srv/app && echo "hello from app-1" > /srv/app/index.html
cat > /etc/systemd/system/demo-app.service <<'UNIT'
[Service]
ExecStart=/usr/bin/python3 -m http.server 8080 --directory /srv/app
Restart=always
[Install]
WantedBy=multi-user.target
UNIT
systemctl daemon-reload && systemctl enable --now demo-app
USERDATA

APP_ID=$(aws ec2 run-instances --image-id $AMI_ID --instance-type t3.micro \
  --subnet-id $APP_A --security-group-ids $SG_APP --metadata-options HttpTokens=required \
  --user-data file:///tmp/app.sh --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=app-1}]' \
  --query 'Instances[0].InstanceId' --output text); save APP_ID

aws ec2 wait instance-running --instance-ids $WEB_ID $APP_ID
WEB_IP=$(aws ec2 describe-instances --instance-ids $WEB_ID --query 'Reservations[0].Instances[0].PublicIpAddress' --output text); save WEB_IP
APP_PRIV=$(aws ec2 describe-instances --instance-ids $APP_ID --query 'Reservations[0].Instances[0].PrivateIpAddress' --output text); save APP_PRIV
echo "web-1 public IP: $WEB_IP    app-1 private IP: $APP_PRIV"
```

---

## 2. The network interface: where networking really happens

Launching created a **network interface (ENI)** for each instance: a virtual network card that lives in the subnet. **The ENI, not the instance, holds the networking settings:**

```console
$ aws ec2 describe-network-interfaces --filters Name=attachment.instance-id,Values=$APP_ID \
    --query 'NetworkInterfaces[0].{ENI:NetworkInterfaceId,Subnet:SubnetId,AZ:AvailabilityZone,PrivateIP:PrivateIpAddress,PublicIP:Association.PublicIp,MAC:MacAddress,SecurityGroups:Groups[].GroupName}'
{
    "ENI": "eni-0a1b2c3d4e5f60718",
    "Subnet": "subnet-0app0a…",
    "AZ": "eu-central-1a",
    "PrivateIP": "10.0.10.37",
    "PublicIP": null,
    "MAC": "02:4f:1a:…",
    "SecurityGroups": ["lab-sg-app"]
}
```

What that means in practice:
- **Security groups are attached to the ENI.** When you "change an instance's security groups", you're changing its primary ENI's.
- An instance always has a **primary ENI** (`eth0`), which can't be removed. You can add **secondary ENIs**, but only from subnets in the **same AZ**. A secondary ENI keeps its IPs and security groups when you move it to another instance, which is a simple failover technique.
- AWS services create ENIs in your subnets too: NAT gateways, load balancers, VPC endpoints, Lambda functions in a VPC, databases. That's why subnets "run out of IPs" even without many servers.
- An ENI only accepts traffic addressed to its own IPs (the **source/destination check**). Turn it off only on servers that forward traffic for others, such as a self-managed NAT or firewall.

### What happens to addresses over time

| Action | Private IP | Auto-assigned public IP | Elastic IP |
|---|---|---|---|
| Reboot | Kept | Kept | Kept |
| **Stop → start** | Kept | **Replaced with a new one** | Kept |
| Terminate | Released | Released | Stays in your account (and keeps costing money) until released |

If something must have a fixed public address, use an Elastic IP, or better, put the server behind a load balancer (Module 05).

---

## 3. Connecting to your servers

| Method | Needs a public IP? | Needs an inbound port open? | Good for |
|---|---|---|---|
| **SSH with a key pair** | Yes (or a VPN) | 22, from your IP only | Quick tests on public servers |
| **EC2 Instance Connect Endpoint** | No | 22, **from the endpoint's security group** only | Private servers, no bastion host, free |
| **SSM Session Manager** | No | **None** | Production: every session is authorized by IAM and logged. Needs the SSM agent (preinstalled on Amazon Linux), an IAM role on the instance, and a path to the SSM service (NAT or endpoints, Module 06) |

An **EC2 Instance Connect Endpoint** is a small AWS-managed entry point you place in a private subnet. Your CLI opens an encrypted tunnel to it over HTTPS (authorized by your IAM identity), and the endpoint forwards the SSH connection to the private server. That's why `sg-app` allows port 22 only from `sg-eice` (Module 03).

---

## 4. Try it: test everything you built

Wait 1–2 minutes for the user-data scripts to finish.

**From your laptop to the public server.** This goes internet → internet gateway → `public-a` → `sg-web`:

```bash
curl http://$WEB_IP          # <h1>web-1 10.0.0.x in eu-central-1a</h1>
```

**Inside the public server:**

```bash
ssh -i ~/lab-key.pem ec2-user@$WEB_IP
  ip -4 addr show ens5 | grep inet     # only the PRIVATE IP 10.0.0.x: the internet gateway translates the public one
  ip route                             # "default via 10.0.0.1": the VPC router at the subnet's .1 address
  curl http://APP_PRIVATE_IP:8080      # use the app IP printed earlier -> "hello from app-1" (sg-app allows sg-web)
  exit
```

**Into the private server, through an Instance Connect Endpoint:**

```bash
EICE_ID=$(aws ec2 create-instance-connect-endpoint --subnet-id $APP_A --security-group-ids $SG_EICE \
  --query InstanceConnectEndpoint.InstanceConnectEndpointId --output text); save EICE_ID
until [ "$(aws ec2 describe-instance-connect-endpoints --instance-connect-endpoint-ids $EICE_ID \
  --query 'InstanceConnectEndpoints[0].State' --output text)" = "create-complete" ]; do sleep 15; done

aws ec2-instance-connect ssh --instance-id $APP_ID --connection-type eice
  curl -s https://checkip.amazonaws.com   # prints the NAT gateway's Elastic IP: outbound internet works
  exit
```

Compare the IP printed by `checkip` with the NAT's Elastic IP:

```bash
aws ec2 describe-addresses --allocation-ids $NAT_EIP --query 'Addresses[0].PublicIp' --output text
```

They match: the private server reaches the internet **as the NAT gateway**, and nothing on the internet can connect to it.

---

## 5. Network performance in one paragraph

Bandwidth depends on the **instance type**. Small types say "up to X Gbps", meaning they can burst above a lower baseline for a while. A single connection tops out around 5 Gbps. Inside a VPC, packets can be up to 9001 bytes ("jumbo frames"), but anything crossing the internet gateway or a VPN is limited to 1500 bytes or less. If small requests work but large transfers hang over a VPN, that's usually an MTU problem (see Module 08's troubleshooting list).

---

## Check yourself

<details><summary>Which launch option decides the instance's AZ and whether it can reach the internet?</summary>The subnet. It fixes the AZ and IP range, and its route table decides internet access.</details>
<details><summary>Where do an instance's security groups and IPs actually live?</summary>On its network interface (ENI).</details>
<details><summary>You stopped and started a server, and its public IP changed. How do you prevent that?</summary>Attach an Elastic IP, or put the server behind a load balancer and use the load balancer's DNS name.</details>
<details><summary>How do you SSH to a server with no public IP, without a bastion host?</summary>An EC2 Instance Connect Endpoint in its VPC, or SSM Session Manager, which needs no inbound port at all.</details>

---
**Previous:** [Module 03](03-security-groups-and-nacls.md) · **Next:** [Module 05 — Load Balancers](05-load-balancers.md)
