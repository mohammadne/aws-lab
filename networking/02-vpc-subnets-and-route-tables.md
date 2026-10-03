# Module 02 — VPC, Subnets & Route Tables

> The skeleton of every AWS network: what a VPC contains, what subnets are for, and exactly where route tables sit and how they get their routes.

---

## 1. The VPC

A **VPC** is an isolated layer-3 network in **one Region**, spanning **all its AZs**. It's defined by one primary IPv4 CIDR (`/16` to `/28`, plus optional secondary and IPv6 CIDRs) and has an **implicit router** that connects all its subnets. A new VPC has **no** path anywhere until you add gateways, routes, and security rules.

```mermaid
%%{init: {"flowchart": {"wrappingWidth": 400}}}%%
flowchart TB
    classDef regional fill:#dbeafe,stroke:#1d4ed8,color:#0b1b3a
    classDef zonal fill:#dcfce7,stroke:#15803d,color:#052e16
    classDef sec fill:#fee2e2,stroke:#b91c1c,color:#450a0a
    classDef gw fill:#ffedd5,stroke:#c2410c,color:#431407
    classDef rt fill:#fef9c3,stroke:#a16207,color:#422006
    classDef compute fill:#ccfbf1,stroke:#0f766e,color:#042f2e
    classDef ext fill:#f1f5f9,stroke:#475569,color:#0f172a
    classDef svc fill:#fef3c7,stroke:#b45309,color:#451a03

    INTERNET(("🌐 Internet")):::ext
    ONPREM["🏢 On-prem network"]:::ext
    OTHERVPC["☁️ Other VPCs"]:::ext
    AWSSVC["🪣 AWS services<br/>S3, DynamoDB"]:::svc

    IGW["🚪 Internet gateway<br/>0..1 per VPC"]:::gw
    EIGW["↗️ Egress-only IGW<br/>IPv6 outbound, 0..1"]:::gw
    VGW["🔒 Virtual private gateway<br/>0..1 attached"]:::gw
    TGWATT["🛰️ Transit Gateway attachment<br/>1 per TGW"]:::gw
    PCX["🔀 VPC peering connections<br/>0..N"]:::gw
    GWE["🛣️ Gateway endpoints<br/>S3 / DynamoDB (route-table based)"]:::gw

    subgraph VPC["☁️ VPC 10.0.0.0/16 + 2600:1f18:abcd:1200::/56 (Region scope, spans all AZs)"]
        ROUTER{{"🧭 Implicit VPC router<br/>reachable at .1 of every subnet"}}:::rt
        subgraph VPCOBJ["VPC-level objects (regional, not inside any subnet)"]
            O1["📋 Route tables<br/>main: exactly 1 · custom: 0..N"]:::rt
            O2["🚧 Network ACLs<br/>default: exactly 1 · custom: 0..N"]:::sec
            O3["🛡️ Security groups<br/>default: exactly 1 · custom: 0..N"]:::sec
            O4["⚙️ DHCP option set: 1<br/>🧭 DNS Resolver at VPC+2"]:::regional
            O5["➕ CIDR blocks: 1 primary + secondary<br/>📒 Flow logs: 0..N"]:::regional
        end
        subgraph AZA["🅰️ AZ-a"]
            subgraph SA1["🟩 public subnet 10.0.0.0/24"]
                NATA["🔁 NAT gateway + EIP"]:::gw
            end
            subgraph SA2["🟦 private subnet 10.0.10.0/24"]
                EC2A["🖥️ EC2 + 🔌 ENI"]:::compute
                IEPA["🔌 Interface endpoint ENI<br/>(PrivateLink to AWS services)"]:::gw
            end
        end
        subgraph AZB["🅱️ AZ-b"]
            subgraph SB1["🟩 public subnet 10.0.1.0/24"]
                ALBB["⚖️ ALB node"]:::gw
            end
            subgraph SB2["🟦 private subnet 10.0.11.0/24"]
                EC2B["🖥️ EC2 + 🔌 ENI"]:::compute
            end
        end
    end

    INTERNET <--> IGW
    INTERNET <--> EIGW
    ONPREM <-->|"VPN / DX"| VGW
    ONPREM <--> TGWATT
    OTHERVPC <--> TGWATT
    OTHERVPC <--> PCX
    AWSSVC <--> GWE
    IGW <--> ROUTER
    EIGW <--> ROUTER
    VGW <--> ROUTER
    TGWATT <--> ROUTER
    PCX <--> ROUTER
    GWE <--> ROUTER
    ROUTER <--> NATA
    ROUTER <--> ALBB
    NATA ~~~ EC2A
    ALBB ~~~ EC2B
    ROUTER <--> EC2A
    ROUTER <--> IEPA
    ROUTER <--> EC2B
    EC2A ~~~ VPCOBJ
```

### What a VPC contains and what you can attach

| Comes with every VPC (exactly 1) | You create inside (0..N) | You attach | Lives **in subnets** (uses IPs) |
|---|---|---|---|
| Implicit router (subnet `.1`) | Subnets | Internet gateway (0–1) | EC2 instances / ENIs |
| Main route table | Custom route tables | Egress-only IGW for IPv6 (0–1) | NAT gateways (zonal) |
| Default NACL (allow all) | Custom NACLs (deny all by default) | Virtual private gateway (0–1) | Interface / GWLB endpoints |
| Default security group | Security groups | Transit Gateway attachment (1 per TGW) | ALB / NLB nodes |
| DHCP option set | Flow logs, extra CIDRs | Peering connections, gateway endpoints | RDS, Lambda, EKS, EFS, … |
| DNS resolver at VPC base + 2 | | Regional NAT gateway | EC2 Instance Connect Endpoint |

### CIDR planning
- Use RFC 1918 ranges. **Never overlap** with on-prem or other VPCs, because overlap breaks peering, TGW, and VPN routing. Use **VPC IPAM** at scale.
- The **primary CIDR can't be changed**, but you can add secondary CIDRs later. `100.64.0.0/10` is a common secondary range for Kubernetes pods.
- Give each AZ an aligned block (e.g. a `/18` per AZ inside a `/16`). Make app subnets large and public subnets small.
- Turn on **`enableDnsHostnames`** (it's off for VPCs created via the API). Private DNS features depend on it.
- **Default VPC** (`172.31.0.0/16`, every subnet public): fine for experiments, not for production.

---

## 2. Subnets

A **subnet** is a slice of the VPC CIDR **pinned to one AZ**. It has four jobs:

1. **Placement:** every ENI lives in a subnet, which decides its AZ and its IP.
2. **Routing:** it's associated with **exactly one route table**.
3. **Filtering:** it's associated with **exactly one network ACL**.
4. **Fault isolation:** one subnet = one AZ, so spreading subnets across AZs gives you HA.

**Is it required?** A VPC can exist without subnets, but **every EC2 instance requires one**, because its primary ENI must live in a subnet. Any service that places ENIs in your VPC (NAT gateway, endpoints, load balancers, RDS, Lambda) also needs subnets.

### Public, private, isolated: decided by the route table

```mermaid
%%{init: {"flowchart": {"wrappingWidth": 400}}}%%
flowchart TB
    classDef zonal fill:#dcfce7,stroke:#15803d,color:#052e16
    classDef gw fill:#ffedd5,stroke:#c2410c,color:#431407
    classDef rt fill:#fef9c3,stroke:#a16207,color:#422006
    classDef ext fill:#f1f5f9,stroke:#475569,color:#0f172a

    NET(("Internet")):::ext
    ONP["On-prem"]:::ext
    IGW["IGW"]:::gw
    VGW["VGW / TGW"]:::gw

    subgraph VPC["VPC 10.0.0.0/16"]
        subgraph PUB["PUBLIC subnet"]
            P["ALB nodes, NAT GW,<br/>bastion (if any)"]:::zonal
        end
        subgraph PRIV["PRIVATE subnet (with NAT)"]
            A["App servers"]:::zonal
        end
        subgraph ISO["ISOLATED subnet"]
            D["Databases"]:::zonal
        end
        subgraph HYB["HYBRID / VPN-only subnet"]
            H["Systems reachable only from on-prem"]:::zonal
        end
        NAT["NAT gateway<br/>(in the public subnet)"]:::gw
        RTPUB["Public RT<br/>10.0.0.0/16 local<br/>0.0.0.0/0 to igw"]:::rt
        RTPRIV["Private RT<br/>10.0.0.0/16 local<br/>0.0.0.0/0 to nat"]:::rt
        RTISO["Isolated RT<br/>10.0.0.0/16 local<br/>(nothing else)"]:::rt
        RTHYB["Hybrid RT<br/>10.0.0.0/16 local<br/>192.168.0.0/16 to vgw/tgw"]:::rt
    end

    RTPUB -.-> PUB
    RTPRIV -.-> PRIV
    RTISO -.-> ISO
    RTHYB -.-> HYB
    PUB <--> IGW <--> NET
    PRIV --> NAT --> IGW
    HYB <--> VGW <--> ONP
```

| Type | Route table has | Reachable from internet | Can reach internet |
|---|---|---|---|
| **Public** | `0.0.0.0/0 → igw` | Yes, if the instance has a public IP and SG/NACL allow | Yes |
| **Private** | `0.0.0.0/0 → nat` | No | Outbound only |
| **Isolated** | `local` only | No | No |

"Auto-assign public IP" doesn't make a subnet public. Only the IGW route does.

**Sizing:** AWS reserves **5 IPs per subnet** (network, `.1` router, `.2` DNS, `.3`, broadcast), so a `/24` has 251 usable and a `/28` has 11. **Subnets can't be resized.** ALB subnets need `/27` or larger. EKS pods consume one IP each.

---

## 3. Route tables

### Where they sit (course question 1)

| Question | Answer |
|---|---|
| **Owned by** | A **VPC** (regional). One table can serve subnets in any AZ |
| **Associated with** | **Subnets**: each subnet has exactly one (explicit, or the VPC's **main** table implicitly). Optionally an **IGW/VGW** ("gateway route table", for inbound inspection) |
| **Assigned to EC2?** | **No, never.** Each ENI uses its subnet's table. Inside the OS, the route is just `default via <subnet>.1` |
| **Role** | The forwarding table of the VPC router for packets **leaving that subnet**. The **source** subnet's table decides the next hop |

```mermaid
%%{init: {"flowchart": {"wrappingWidth": 400}}}%%
flowchart TB
    classDef pub fill:#dcfce7,stroke:#15803d,color:#052e16
    classDef priv fill:#dbeafe,stroke:#1d4ed8,color:#0b1b3a
    classDef gw fill:#ffedd5,stroke:#c2410c,color:#431407
    classDef rt fill:#fef9c3,stroke:#a16207,color:#422006
    classDef ext fill:#f1f5f9,stroke:#475569,color:#0f172a
    classDef compute fill:#ccfbf1,stroke:#0f766e,color:#042f2e
    classDef db fill:#e0e7ff,stroke:#4338ca,color:#1e1b4b

    NET(("🌐 Internet<br/>👥 users")):::ext

    PUBRT["📋 rtb-public<br/>associated: public-a, public-b<br/>──────────────<br/>10.10.0.0/16 → local<br/>0.0.0.0/0 → igw-0a1b"]:::rt
    PRIVA["📋 rtb-private-a<br/>associated: app-a<br/>──────────────<br/>10.10.0.0/16 → local<br/>0.0.0.0/0 → nat-a"]:::rt
    MAINRT["📋 MAIN route table<br/>implicit: data-a, data-b<br/>──────────────<br/>10.10.0.0/16 → local"]:::rt
    PRIVB["📋 rtb-private-b<br/>associated: app-b<br/>──────────────<br/>10.10.0.0/16 → local<br/>0.0.0.0/0 → nat-b"]:::rt

    subgraph REGION["🗺️ Region us-east-1"]
        subgraph VPC["☁️ VPC 10.10.0.0/16"]
            IGW["🚪 Internet gateway igw-0a1b<br/>(regional, 1:1 NAT for public IPs)"]:::gw
            subgraph AZA["🅰️ Availability Zone A"]
                subgraph PUBA["🟩 public-a 10.10.0.0/24"]
                    ELBA["⚖️ ALB node<br/>10.10.0.20"]:::gw
                    NATA["🔁 NAT GW nat-a<br/>10.10.0.50 + EIP"]:::gw
                end
                subgraph APPA["🟦 app-a 10.10.2.0/24 (private)"]
                    EC2A["🖥️ EC2 web/app<br/>ENI 10.10.2.15<br/>🛡️ sg-app"]:::compute
                end
                subgraph DBA["🟦 data-a 10.10.11.0/24 (isolated)"]
                    RDSA["🗄️ DB writer (M)<br/>🛡️ sg-db"]:::db
                end
            end
            subgraph AZB["🅱️ Availability Zone B"]
                subgraph PUBB["🟩 public-b 10.10.1.0/24"]
                    ELBB["⚖️ ALB node<br/>10.10.1.20"]:::gw
                    NATB["🔁 NAT GW nat-b<br/>10.10.1.50 + EIP"]:::gw
                end
                subgraph APPB["🟦 app-b 10.10.3.0/24 (private)"]
                    EC2B["🖥️ EC2 web/app<br/>ENI 10.10.3.27<br/>🛡️ sg-app"]:::compute
                end
                subgraph DBB["🟦 data-b 10.10.12.0/24 (isolated)"]
                    RDSB["🗄️ DB standby / reader (S)<br/>⇠ replication from M"]:::db
                end
            end
        end
    end

    NET <-->|"① inbound HTTPS"| IGW
    IGW --> ELBA
    IGW --> ELBB
    ELBA -->|"to target"| EC2A
    ELBB -->|"to target"| EC2B
    NATA <-->|"② egress via NAT"| EC2A
    NATB <-->|"② egress via NAT"| EC2B
    IGW <--> NATA
    IGW <--> NATB
    EC2A -->|"SQL"| RDSA
    EC2B -->|"SQL"| RDSB
    PUBRT -.- PUBA
    PUBRT -.- PUBB
    PRIVA -.- APPA
    PRIVB -.- APPB
    MAINRT -.- DBA
    MAINRT -.- DBB

    linkStyle 0,1,2,3,4 stroke:#dc2626,stroke-width:2px
    linkStyle 5,6,7,8 stroke:#16a34a,stroke-width:2px
    linkStyle 11,12,13,14,15,16 stroke:#a16207,stroke-width:1px,stroke-dasharray:4
```

### How a route table is populated

| Source | Example | Notes |
|---|---|---|
| ① **Automatic local route** | `10.10.0.0/16 → local` | One per VPC CIDR. Can't be deleted. All subnets can reach each other by default |
| ② **Static routes** (you) | `0.0.0.0/0 → igw/nat`, `10.1.0.0/16 → pcx`, `10.0.0.0/8 → tgw` | Most routes |
| ③ **Propagated** from a VGW | On-prem prefixes over VPN/Direct Connect BGP | Turn on route propagation per table |
| ④ **Gateway endpoints** | `pl-xxxx (S3) → vpce-xxxx` | Added automatically to the tables you select |
| ⑤ **VPC Route Server** | Prefixes advertised by appliances over BGP | For appliance failover |

Transit Gateway **never** writes into VPC route tables. You add `… → tgw-…` routes yourself.

### How a route is chosen
1. **Longest prefix match** wins (IPv4 and IPv6 are evaluated separately).
2. For the same prefix: **static beats propagated**. Among VGW-propagated routes: Direct Connect, then VPN static, then VPN BGP.
3. If the target was deleted, the route shows **blackhole** and its traffic is dropped.
4. There's no ECMP, no source-based routing, and **no transitive routing** through peering or a gateway endpoint.

**Common targets:** `local`, `igw-`, `eigw-` (IPv6 out), `nat-`, `vgw-`, `tgw-`, `pcx-`, `vpce-` (gateway or GWLB endpoint), `eni-` (appliance).

**Best practice:** keep the **main** table local-only, so a subnet you forget to associate stays private.

---

## 4. Hands-on: build the VPC

> These labs build one environment across Modules 02–08. They create billable resources (mostly the NAT gateway in Module 03 and public IPv4 addresses). Set a **budget alert**, and run the **cleanup in Module 08**.

**Setup (once):** you need AWS CLI v2 and an admin identity (not root). Run everything in **bash**, and save variables so you can resume later.

```bash
bash
aws sts get-caller-identity
export AWS_REGION=us-east-1 AWS_DEFAULT_REGION=us-east-1
save() { echo "export $1=\"${!1}\"" >> ~/aws-lab.env; }   # new shell later? run bash, re-run this line, then: source ~/aws-lab.env
```

**Build:** a VPC, 6 subnets (3 tiers × 2 AZs), an IGW, and route tables.

```bash
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=lab-vpc}]' --query Vpc.VpcId --output text); save VPC_ID
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames '{"Value":true}'

AZ1=$(aws ec2 describe-availability-zones --filters Name=zone-type,Values=availability-zone --query 'AvailabilityZones[0].ZoneName' --output text); save AZ1
AZ2=$(aws ec2 describe-availability-zones --filters Name=zone-type,Values=availability-zone --query 'AvailabilityZones[1].ZoneName' --output text); save AZ2

mk_subnet() { aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block $2 --availability-zone $3 \
  --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=$1}]" --query Subnet.SubnetId --output text; }
PUB_A=$(mk_subnet public-a 10.0.0.0/24 $AZ1);   save PUB_A
PUB_B=$(mk_subnet public-b 10.0.1.0/24 $AZ2);   save PUB_B
APP_A=$(mk_subnet app-a 10.0.10.0/24 $AZ1);     save APP_A
APP_B=$(mk_subnet app-b 10.0.11.0/24 $AZ2);     save APP_B
DATA_A=$(mk_subnet data-a 10.0.20.0/24 $AZ1);   save DATA_A
DATA_B=$(mk_subnet data-b 10.0.21.0/24 $AZ2);   save DATA_B
aws ec2 modify-subnet-attribute --subnet-id $PUB_A --map-public-ip-on-launch
aws ec2 modify-subnet-attribute --subnet-id $PUB_B --map-public-ip-on-launch

IGW_ID=$(aws ec2 create-internet-gateway --query InternetGateway.InternetGatewayId --output text); save IGW_ID
aws ec2 attach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID

RT_PUB=$(aws ec2 create-route-table --vpc-id $VPC_ID --query RouteTable.RouteTableId --output text); save RT_PUB
aws ec2 create-route --route-table-id $RT_PUB --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID
aws ec2 associate-route-table --route-table-id $RT_PUB --subnet-id $PUB_A
aws ec2 associate-route-table --route-table-id $RT_PUB --subnet-id $PUB_B

RT_PRIV=$(aws ec2 create-route-table --vpc-id $VPC_ID --query RouteTable.RouteTableId --output text); save RT_PRIV
aws ec2 associate-route-table --route-table-id $RT_PRIV --subnet-id $APP_A
aws ec2 associate-route-table --route-table-id $RT_PRIV --subnet-id $APP_B
# data-a / data-b: no association, so they use the MAIN table
```

✅ **Check which route table each subnet uses:**

```bash
for s in $PUB_A $PUB_B $APP_A $APP_B $DATA_A $DATA_B; do
  rt=$(aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=$s --query 'RouteTables[0].RouteTableId' --output text)
  echo "$s -> ${rt/None/MAIN (implicit)}"
done
```

---

## Check yourself

<details><summary>Route table: assigned to EC2, subnet, or VPC?</summary>Owned by the VPC, associated with subnets (the main table by default), optionally with an IGW/VGW. Never with EC2.</details>
<details><summary>What makes a subnet public?</summary>Its route table has a route to an internet gateway.</details>
<details><summary>A route to 0.0.0.0/0 via NAT and a gateway endpoint for S3. Which one carries S3 traffic?</summary>The endpoint. Its prefix-list routes are more specific than /0.</details>
<details><summary>VPC A peers with B, and B has a NAT gateway. Can A use it?</summary>No. Peering has no transitive or edge-to-edge routing.</details>

---
**Previous:** [Module 01](01-foundations.md) · **Next:** [Module 03 — Internet Access](03-internet-access.md)
