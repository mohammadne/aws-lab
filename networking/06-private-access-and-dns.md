# Module 06 — Private Access & DNS: Endpoints, PrivateLink, Route 53 Resolver

> Reach AWS services and other teams' services without the internet or NAT, and understand how name resolution works inside a VPC and to and from on-prem.

---

## 1. VPC endpoints

| | **Gateway endpoint** | **Interface endpoint** (PrivateLink) | **GWLB endpoint** |
|---|---|---|---|
| Services | **S3, DynamoDB only** | Most AWS services, SaaS, your own services | Your firewall fleet behind a GWLB |
| Mechanism | **Route table** entry `pl-… → vpce-…` | **ENI** with a private IP in each chosen subnet, plus **private DNS** | Route table target, GENEVE to appliances |
| Security group | ❌ (use the endpoint policy) | ✅ | ❌ |
| Usable from on-prem / peered VPCs | ❌ | ✅ | via routing |
| Cost | **Free** | Per AZ-hour + per GB | Per hour + per GB |

### Gateway endpoint: routing-based

```mermaid
%%{init: {"flowchart": {"wrappingWidth": 240}}}%%
flowchart LR
    classDef zonal fill:#dcfce7,stroke:#15803d,color:#052e16
    classDef rt fill:#fef9c3,stroke:#a16207,color:#422006
    classDef gw fill:#ffedd5,stroke:#c2410c,color:#431407
    classDef svc fill:#fef3c7,stroke:#b45309,color:#451a03
    classDef compute fill:#ccfbf1,stroke:#0f766e,color:#042f2e

    subgraph VPC["☁️ VPC 10.0.0.0/16"]
        subgraph PRIV["🟦 private subnet app-a"]
            EC2["🖥️ EC2<br/>aws s3 cp …"]:::compute
        end
        RT["📋 rtb-app-a<br/>10.0.0.0/16 → local<br/>0.0.0.0/0 → nat-a<br/>pl-63a5400a (S3 prefixes) → vpce-0s3<br/>(added automatically when you<br/>select this RT on the endpoint)"]:::rt
        GWE["🛣️ vpce-0s3 (Gateway, S3)<br/>📜 endpoint policy:<br/>allow only bucket my-app-*"]:::gw
    end
    S3["🪣 Amazon S3 (same Region)<br/>bucket policy can require<br/>aws:SourceVpce = vpce-0s3"]:::svc

    RT -.- PRIV
    EC2 -->|"dst 52.216.x.x matches pl-…"| GWE --> S3
```

DNS doesn't change: the S3 hostname still resolves to public S3 IPs. The **route** is what changes, because the prefix-list route beats `0/0 → nat`. Lock buckets to your VPC with `aws:SourceVpce` conditions in the bucket policy.

### Interface endpoint: ENI + DNS based

```mermaid
%%{init: {"flowchart": {"wrappingWidth": 400}}}%%
flowchart TB
    classDef zonal fill:#dcfce7,stroke:#15803d,color:#052e16
    classDef gw fill:#ffedd5,stroke:#c2410c,color:#431407
    classDef svc fill:#fef3c7,stroke:#b45309,color:#451a03
    classDef compute fill:#ccfbf1,stroke:#0f766e,color:#042f2e
    classDef global fill:#f3e8ff,stroke:#7e22ce,color:#1f1147
    classDef ext fill:#f1f5f9,stroke:#475569,color:#0f172a

    ONP["🏢 On-prem via DX/VPN<br/>(can use interface endpoints)"]:::ext
    subgraph VPC["☁️ VPC 10.0.0.0/16 (enableDnsHostnames + enableDnsSupport = true)"]
        DNS["🧭 Private DNS (hidden PHZ)<br/>ssm.us-east-1.amazonaws.com →<br/>10.0.10.100, 10.0.11.100"]:::global
        subgraph AZA["🅰️ AZ-a"]
            subgraph SA["🟦 app-a 10.0.10.0/24"]
                EA["🖥️ EC2"]:::compute
                ENIA["🔌 endpoint ENI 10.0.10.100<br/>🛡️ sg-endpoints: 443 from VPC"]:::gw
            end
        end
        subgraph AZB["🅱️ AZ-b"]
            subgraph SB["🟦 app-b 10.0.11.0/24"]
                EB["🖥️ EC2"]:::compute
                ENIB["🔌 endpoint ENI 10.0.11.100<br/>🛡️ sg-endpoints"]:::gw
            end
        end
    end
    SVC["🛠️ AWS Systems Manager<br/>(regional service)"]:::svc

    EA -->|"① resolve ssm.us-east-1…"| DNS
    EA -->|"② HTTPS to 10.0.10.100"| ENIA
    EB --> ENIB
    ENIA -->|"③ PrivateLink (AWS network)"| SVC
    ENIB --> SVC
    ONP -.->|"via inbound Resolver endpoint + routing"| ENIA
```

- Pick **one subnet per AZ**. Private DNS makes the **normal service hostname** resolve to the endpoint's IPs, so you need **no code changes**.
- Requires `enableDnsSupport` and `enableDnsHostnames`, plus an endpoint SG allowing **443 from your clients**.
- **Common sets:** SSM without NAT = `ssm`, `ssmmessages`, `ec2messages`. ECR without NAT = `ecr.api`, `ecr.dkr` **+ the S3 gateway endpoint**.
- In multi-VPC estates, centralize endpoints in a shared-services VPC to save per-AZ-hour charges.

### PrivateLink for your own services

```mermaid
%%{init: {"flowchart": {"wrappingWidth": 240}}}%%
flowchart LR
    classDef compute fill:#ccfbf1,stroke:#0f766e,color:#042f2e
    classDef gw fill:#ffedd5,stroke:#c2410c,color:#431407
    classDef regional fill:#dbeafe,stroke:#1d4ed8,color:#0b1b3a

    subgraph CONSUMER["👤 Consumer account: VPC 10.0.0.0/16"]
        CEC2["🖥️ Client EC2"]:::compute
        CEP["🔌 Interface endpoint ENIs<br/>10.0.5.10 / 10.0.6.10<br/>🛡️ SG"]:::gw
        CEC2 --> CEP
    end
    subgraph PROVIDER["🏬 Provider account: VPC 10.0.0.0/16 (overlap is OK!)"]
        SVC["📜 Endpoint service<br/>com.amazonaws.vpce.us-east-1.vpce-svc-0abc<br/>acceptance required, allowed principals"]:::regional
        NLB["⚖️ Network Load Balancer<br/>(or GWLB)"]:::gw
        APP["🖥️ Service fleet"]:::compute
        SVC --- NLB --> APP
    end
    CEP ==>|"PrivateLink: one-way,<br/>consumer to provider only"| SVC
```

Use it to expose **one service** to other VPCs or accounts. It's **one-way** (consumer → provider) and **overlapping CIDRs don't matter**. Compare that with peering, which exposes whole networks.

---

## 2. DNS inside a VPC

- The **Route 53 Resolver** answers at **VPC base + 2** (e.g. `10.0.0.2`) and at `169.254.169.253`. It needs `enableDnsSupport`.
- It resolves public DNS, EC2 names (`ip-10-0-1-5.ec2.internal`), **private hosted zones** associated with the VPC, and endpoint private DNS.
- Public EC2 hostnames are **split-horizon**: they resolve to the private IP inside the VPC and the public IP outside (needs `enableDnsHostnames`).
- Limit: 1,024 packets/s per ENI to the Resolver. Cache in chatty apps. DNS traffic **isn't** filtered by SGs and doesn't appear in Flow Logs (use Resolver query logs).

```mermaid
%%{init: {"flowchart": {"wrappingWidth": 400}}}%%
flowchart TB
    classDef q fill:#fef9c3,stroke:#a16207,color:#422006
    classDef ok fill:#dcfce7,stroke:#15803d,color:#052e16
    classDef ext fill:#f1f5f9,stroke:#475569,color:#0f172a
    classDef gw fill:#ffedd5,stroke:#c2410c,color:#431407

    Q(["🖥️ Query from an instance to VPC+2"]):::q
    RULE{"Matching Resolver FORWARD rule?<br/>(most specific domain wins)"}:::q
    PHZ{"Matching PRIVATE HOSTED ZONE<br/>associated with this VPC?"}:::q
    INT{"VPC-internal name?<br/>ip-10-0-1-5.ec2.internal,<br/>endpoint private DNS"}:::q
    PUB["🌍 Recursive resolution<br/>on the public internet DNS"]:::ext
    FWD["➡️ Forward to target IPs<br/>(e.g. on-prem DNS) via an<br/>OUTBOUND endpoint"]:::gw
    ANS1["Answer from the private zone"]:::ok
    ANS2["Answer from VPC records"]:::ok

    Q --> RULE
    RULE -->|"yes"| FWD
    RULE -->|"no"| PHZ
    PHZ -->|"yes"| ANS1
    PHZ -->|"no"| INT
    INT -->|"yes"| ANS2
    INT -->|"no"| PUB
```

### Hybrid DNS

```mermaid
%%{init: {"flowchart": {"wrappingWidth": 400}}}%%
flowchart TB
    classDef zonal fill:#dcfce7,stroke:#15803d,color:#052e16
    classDef gw fill:#ffedd5,stroke:#c2410c,color:#431407
    classDef ext fill:#f1f5f9,stroke:#475569,color:#0f172a
    classDef compute fill:#ccfbf1,stroke:#0f766e,color:#042f2e
    classDef global fill:#f3e8ff,stroke:#7e22ce,color:#1f1147

    subgraph ONP["🏢 On-prem"]
        ODNS["On-prem DNS servers<br/>10.200.0.53 / 10.200.1.53<br/>zone corp.example.com"]:::ext
        OCL["💻 On-prem clients"]:::ext
    end

    subgraph VPC["☁️ VPC 10.0.0.0/16"]
        R2["🧭 Route 53 Resolver 10.0.0.2"]:::global
        INB["📥 INBOUND endpoint<br/>ENIs 10.0.1.53 (AZ-a), 10.0.2.53 (AZ-b)<br/>🛡️ SG: 53 TCP/UDP from on-prem"]:::gw
        OUTB["📤 OUTBOUND endpoint<br/>ENIs in 2 AZs<br/>🛡️ SG: 53 out to on-prem DNS"]:::gw
        RULE["📜 Forwarding rule<br/>corp.example.com → 10.200.0.53, 10.200.1.53<br/>associated with VPCs (shareable via RAM)"]:::gw
        EC2["🖥️ EC2 app"]:::compute
        PHZ["Private hosted zone aws.example.com"]:::global
    end

    EC2 -->|"① app.corp.example.com?"| R2
    R2 -->|"② rule matches"| RULE
    RULE --> OUTB
    OUTB -->|"③ over DX / VPN"| ODNS

    OCL -->|"A: db.aws.example.com?"| ODNS
    ODNS -->|"B: conditional forwarder<br/>aws.example.com → inbound IPs"| INB
    INB --> R2
    R2 -->|"C: answer from the PHZ"| PHZ
```

- On-prem **can't** query `.2` over VPN/Direct Connect. Give it a **Resolver inbound endpoint**.
- AWS → on-prem lookups need an **outbound endpoint + forwarding rule** (`corp.example.com → on-prem DNS`). Rules can be shared across accounts with RAM.
- **DHCP option sets** are immutable: to change one, create a new set and associate it. If you point instances at your own DNS servers, those servers must forward AWS names to `.2`, or private zones and endpoints break.

---

## 3. Hands-on: gateway endpoint, then remove the NAT

```bash
aws ec2 describe-route-tables --route-table-ids $RT_PRIV --query 'RouteTables[0].Routes'   # before
S3_EP=$(aws ec2 create-vpc-endpoint --vpc-id $VPC_ID --vpc-endpoint-type Gateway \
  --service-name com.amazonaws.$AWS_REGION.s3 --route-table-ids $RT_PRIV \
  --query VpcEndpoint.VpcEndpointId --output text); save S3_EP
aws ec2 describe-route-tables --route-table-ids $RT_PRIV --query 'RouteTables[0].Routes'   # after: pl-… -> vpce-…

# Remove internet egress: S3 keeps working privately, the internet doesn't
aws ec2 delete-route --route-table-id $RT_PRIV --destination-cidr-block 0.0.0.0/0
aws ec2-instance-connect ssh --instance-id $APP_ID --connection-type eice
  curl -m 5 -s https://checkip.amazonaws.com || echo "internet: blocked"
  curl -m 5 -s -o /dev/null -w "S3 HTTP %{http_code}\n" https://s3.us-east-1.amazonaws.com/   # use your Region
  exit

# Stop paying for the NAT gateway
aws ec2 delete-nat-gateway --nat-gateway-id $NAT_ID
until [ "$(aws ec2 describe-nat-gateways --nat-gateway-ids $NAT_ID --query 'NatGateways[0].State' --output text)" = "deleted" ]; do sleep 15; done
aws ec2 release-address --allocation-id $NAT_EIP
```

Any HTTP status from S3 (200, 307, or 403) proves the private path works.

---

## Check yourself

<details><summary>Gateway vs interface endpoint: how does each steer traffic?</summary>Gateway: a route table entry (prefix list → vpce). Interface: an ENI with a private IP, plus private DNS for the service name.</details>
<details><summary>On-prem needs private S3 access over Direct Connect. Which endpoint?</summary>An S3 interface endpoint. Gateway endpoints only work from inside the VPC.</details>
<details><summary>Can on-prem servers use 10.0.0.2 for DNS over a VPN?</summary>No. Use a Resolver inbound endpoint.</details>

---
**Previous:** [Module 05](05-ec2-networking-and-load-balancers.md) · **Next:** [Module 07 — Connecting Networks](07-connecting-networks.md)
