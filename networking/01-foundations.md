# Module 01 — Foundations: How AWS Networking Works

← [All tutorials](../README.md) · **AWS Networking tutorial**, module 1 of 8

## Tutorial overview

A compact, diagram-driven course for developers and network engineers who need to **understand how AWS networking works internally** and **make sound design decisions** about VPCs, subnets, route tables, gateways, security groups, EC2, endpoints, and hybrid connectivity. It has 8 modules, each ending with a short hands-on AWS CLI lab that builds on the previous one.

### Course map

```mermaid
%%{init: {"flowchart": {"wrappingWidth": 400}}}%%
flowchart TB
    classDef found fill:#f3e8ff,stroke:#7e22ce,color:#1f1147
    classDef core fill:#dbeafe,stroke:#1d4ed8,color:#0b1b3a
    classDef sec fill:#fee2e2,stroke:#b91c1c,color:#450a0a
    classDef conn fill:#ffedd5,stroke:#c2410c,color:#431407
    classDef ops fill:#dcfce7,stroke:#15803d,color:#052e16

    subgraph P1["Build the network"]
        direction LR
        M01["01 🌍 Foundations<br/>scopes, overlay, costs"]:::found
        M02["02 ☁️ VPC, subnets<br/>& route tables"]:::core
        M03["03 🚪 Internet access<br/>IGW, NAT, public IPs"]:::core
        M04["04 🛡️ Security groups<br/>& NACLs"]:::sec
        M01 --> M02 --> M03 --> M04
    end
    subgraph P2["Run workloads and connect"]
        direction LR
        M05["05 🖥️ EC2 networking<br/>& load balancers"]:::core
        M06["06 🔌 Endpoints,<br/>PrivateLink & DNS"]:::conn
        M07["07 🛰️ Peering, TGW,<br/>VPN, Direct Connect"]:::conn
        M08["08 🔎 Troubleshooting, costs,<br/>big picture, review"]:::ops
        M05 --> M06 --> M07 --> M08
    end
    P1 --> P2
```

| # | Module | You'll learn |
|---|---|---|
| 01 | [Foundations](01-foundations.md) | Network concepts mapped to AWS; global/regional/zonal scopes; how the VPC overlay works; cost basics |
| 02 | [VPC, subnets & route tables](02-vpc-subnets-and-route-tables.md) | What a VPC contains and accepts; the subnet's role; where route tables attach and how they're populated |
| 03 | [Internet access](03-internet-access.md) | IGW vs NAT gateway, public IP types, HA egress design |
| 04 | [Security groups & NACLs](04-security-groups-and-nacls.md) | Stateful vs stateless filtering, the evaluation pipeline, SG chaining |
| 05 | [EC2 networking & load balancers](05-ec2-networking-and-load-balancers.md) | Launch components, ENIs, IP lifecycle, IAM roles, safe access, ALB vs NLB |
| 06 | [Private access & DNS](06-private-access-and-dns.md) | Gateway vs interface endpoints, PrivateLink, Route 53 Resolver, hybrid DNS |
| 07 | [Connecting networks](07-connecting-networks.md) | Peering vs Transit Gateway vs PrivateLink, VPN, Direct Connect, route preference |
| 08 | [Operate & review](08-operate-and-review.md) | Troubleshooting, cleanup, cost traps, the full architecture, decision cheat sheet, quick answers |

### The five questions this tutorial answers

1. **Is a route table assigned to EC2, a subnet, or the VPC? What's its role, and how is it populated?** → [Module 02 §3](02-vpc-subnets-and-route-tables.md#3-route-tables)
2. **What does a VPC contain, and what can you attach to it?** → [Module 02 §1](02-vpc-subnets-and-route-tables.md#1-the-vpc)
3. **What's the role of a subnet, and is it required?** → [Module 02 §2](02-vpc-subnets-and-route-tables.md#2-subnets)
4. **What must you assign when launching EC2?** → [Module 05 §1](05-ec2-networking-and-load-balancers.md#1-what-you-assign-at-launch-course-question-4)
5. **What's a security group? Can you skip it? Is it attached to the instance, the subnet, or the VPC?** → [Module 04 §3](04-security-groups-and-nacls.md#3-security-groups-what-you-must-know)

All five answers are summarized together in [Module 08 §6](08-operate-and-review.md#6-the-five-questions-quick-answers).

> [!WARNING]
> The labs create billable resources (a NAT gateway, public IPv4 addresses, flow logs). Set a budget alert and run the cleanup in [Module 08](08-operate-and-review.md#2-hands-on-break-it-diagnose-it-then-clean-up).

*Reflects AWS as of 2025–2026 (Regional NAT gateway, 5 Gbps VPN tunnels, VPC Route Server, Block Public Access, hourly public IPv4 pricing). Confirm quotas and prices for your Region.*

---

> Know where every resource lives (global, regional, or zonal) and how the VPC network behaves under the hood. Everything later builds on this.

---

## 1. Your networking knowledge, translated

| On-prem concept | AWS equivalent | What's different |
|---|---|---|
| Routing domain / private network | **VPC** | Regional, defined by CIDRs, isolated by default |
| VLAN / L2 segment | **Subnet** | One AZ only. No broadcast, no multicast, ARP answered by the hypervisor |
| Default gateway / core router | **Implicit VPC router** (subnet `.1`) | Invisible and distributed. You only control its **route tables** |
| Routing table | **Route table** | Associated with **subnets**, not devices |
| VRF | **Transit Gateway route table** | VPC route tables aren't VRFs |
| Stateful host firewall | **Security group** | Per network interface (ENI), allow-only |
| Router ACL | **Network ACL** | Per subnet, stateless, ordered |
| Edge router + NAT | **Internet gateway** | Also does 1:1 NAT to public IPs |
| PAT / NAT overload | **NAT gateway** | Managed, outbound only |
| IPsec / MPLS / leased line | **Site-to-Site VPN / Direct Connect** | BGP over IPsec tunnels or 802.1Q VLANs |
| NIC | **ENI** (Elastic Network Interface) | Holds IPs, MAC, SGs. Can move between instances in the same AZ |
| NetFlow / SPAN | **VPC Flow Logs / Traffic Mirroring** | Flow metadata only / VXLAN packet copies |
| VRRP / HSRP | ❌ Not available | Fail over by moving an ENI or IP, changing a route, or using a load balancer |

---

## 2. Regions, AZs, and resource scopes

- A **Region** (`eu-central-1`) is fully independent, with its own APIs and resources. A **VPC never spans Regions.**
- An **Availability Zone** is one or more data centers with independent power and networking. **The AZ is your failure domain.**
- AZ **names** (`us-east-1a`) are shuffled per account. AZ **IDs** (`use1-az4`) are identical everywhere, so use IDs across accounts.
- Local Zones, Wavelength Zones, and Outposts extend a VPC by adding a **subnet** in that location.

```mermaid
%%{init: {"flowchart": {"wrappingWidth": 400}}}%%
flowchart TB
    classDef global fill:#f3e8ff,stroke:#7e22ce,color:#1f1147
    classDef regional fill:#dbeafe,stroke:#1d4ed8,color:#0b1b3a
    classDef zonal fill:#dcfce7,stroke:#15803d,color:#052e16
    classDef sec fill:#fee2e2,stroke:#b91c1c,color:#450a0a
    classDef gw fill:#ffedd5,stroke:#c2410c,color:#431407
    classDef rt fill:#fef9c3,stroke:#a16207,color:#422006
    classDef compute fill:#ccfbf1,stroke:#0f766e,color:#042f2e

    subgraph G["🟪 GLOBAL: one per account"]
        G1["🪪 IAM roles, users, policies,<br/>instance profiles"]:::global
        G2["🧭 Route 53 hosted zones<br/>(public + private)"]:::global
        G3["🌍 CloudFront<br/>🚀 Global Accelerator"]:::global
        G4["🌐 Direct Connect gateway<br/>🕸️ Cloud WAN"]:::global
    end

    subgraph R["🟦 REGIONAL: usable from every AZ in the Region"]
        subgraph RN["Network containers and policy"]
            R1["☁️ VPC + CIDRs"]:::regional
            R2["📋 Route tables"]:::rt
            R3["🛡️ Security groups"]:::sec
            R4["🚧 Network ACLs"]:::sec
            R5["⚙️ DHCP option sets<br/>📑 prefix lists"]:::regional
        end
        subgraph RG["Gateways and connections"]
            R6["🚪 IGW<br/>↗️ egress-only IGW"]:::gw
            R7["🔒 VGW · 📇 CGW<br/>🔐 VPN"]:::gw
            R8["🛰️ Transit Gateway<br/>🔀 peering"]:::gw
            R9["🛣️ Gateway endpoints"]:::gw
            R10["🔁 Regional NAT GW<br/>📍 Elastic IPs"]:::gw
        end
        subgraph RC["Compute and storage"]
            R12["💿 AMIs · 🔑 key pairs<br/>📄 launch templates"]:::regional
            R13["📈 Auto Scaling groups"]:::regional
            R14["📸 EBS snapshots<br/>🪣 S3 buckets"]:::regional
            R15["⚖️ Load balancers *"]:::regional
            R16["🔌 Interface endpoints *<br/>🛰️ TGW attachments *"]:::regional
        end
    end

    subgraph Z["🟩 ZONAL: lives in exactly one AZ (anything with an ENI)"]
        Z1["🟦 Subnet"]:::zonal
        Z2["🖥️ EC2 instance"]:::compute
        Z3["🔌 ENI"]:::zonal
        Z4["💽 EBS volume"]:::zonal
        Z5["🔁 NAT GW (zonal)"]:::gw
        Z6["⚖️ LB nodes · 🔌 endpoint ENIs<br/>🛰️ TGW attachment ENIs"]:::zonal
        Z7["📦 Capacity reservation<br/>🗄️ dedicated host"]:::zonal
    end

    G ~~~ R
    RN ~~~ RG ~~~ RC
    R ~~~ Z
    R15 -.->|"* has per-AZ parts"| Z6
    R16 -.-> Z6
```

**Rules of thumb**
- If it has an IP address on the wire, it's **zonal**. Policy and container objects are **regional**. Identity and DNS are **global**.
- Zonal things only combine **within the same AZ**: an EBS volume or ENI can only attach to an instance in its own AZ.
- Regional objects are usable from every AZ. One security group or route table can serve subnets in all AZs.

---

## 3. Under the hood: the VPC is an overlay

When you change the network, you call a **regional API** (control plane). The change is pushed to the **Nitro card** on every host, and the Nitro cards enforce routes, SGs, and NACLs and encapsulate packets (data plane). There is **no central router or firewall box**.

```mermaid
sequenceDiagram
    autonumber
    participant A as Instance A<br/>10.0.1.10 (AZ-a)
    participant NA as Nitro card on host A
    participant M as Mapping service
    participant NB as Nitro card on host B
    participant B as Instance B<br/>10.0.2.20 (AZ-b)

    A->>NA: ARP who-has 10.0.1.1 (gateway)?
    NA-->>A: Reply from Nitro (proxy ARP). There is no real L2 broadcast.
    A->>NA: IP packet src 10.0.1.10 dst 10.0.2.20
    Note over NA: 1. Outbound SG check (stateful)<br/>2. Subnet NACL outbound check<br/>3. Route table lookup: 10.0.0.0/16 local
    NA->>M: Where is 10.0.2.20 in this VPC? (cached)
    M-->>NA: On physical host B
    NA->>NB: Encapsulated packet over the AWS physical network
    Note over NB: 4. Destination subnet NACL inbound<br/>5. Destination SG inbound
    NB->>B: Original packet delivered
```

**Consequences you'll actually hit:**
- **No broadcast or multicast** (except via Transit Gateway multicast). **No promiscuous sniffing**: use Traffic Mirroring instead.
- **No VRRP or gratuitous-ARP failover.** For HA, use a load balancer, move a secondary IP or ENI, or update a route target.
- **Source/destination check** drops traffic that isn't to or from the ENI's own IPs. Disable it on NAT instances, firewalls, and routers.
- Changes are **eventually consistent**. In scripts, use `aws ec2 wait …` or poll, rather than assuming instant effect.
- Running traffic keeps flowing even if the control plane has issues (**static stability**).

---

## 4. Cost basics that drive design

| Traffic | Cost (check pricing for your Region) |
|---|---|
| Same AZ, private IPs | Free |
| Cross-AZ | ~$0.01/GB **each direction** |
| Via a NAT gateway | Hourly + per GB processed |
| Any public IPv4 (in use or idle) | ~$0.005/hour each |
| Internet egress | Per GB (ingress is free) |

**Design implication:** keep chatty traffic inside an AZ where that's safe, always use private IPs inside AWS, use one NAT per AZ, and use endpoints for AWS services.

---

## Check yourself

<details><summary>Is a subnet regional or zonal? A security group?</summary>A subnet is zonal (one AZ). A security group is regional (VPC-scoped) and usable in every AZ.</details>
<details><summary>Can an EBS volume in AZ-a attach to an instance in AZ-b?</summary>No. Both are zonal. Snapshot the volume and restore it in AZ-b.</details>
<details><summary>Where are security group rules enforced?</summary>On the Nitro card of each instance's host. Enforcement is distributed, so there's no central firewall.</details>

---
**Next:** [Module 02 — VPC, Subnets & Route Tables](02-vpc-subnets-and-route-tables.md)
