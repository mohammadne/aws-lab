# Module 07 — Connecting Networks: Peering, Transit Gateway, VPN, Direct Connect

> Connect VPCs to each other and to your data centers, and pick the right option for the job.

---

## 1. Choosing

```mermaid
%%{init: {"flowchart": {"wrappingWidth": 400}}}%%
flowchart TB
    classDef q fill:#fef9c3,stroke:#a16207,color:#422006
    classDef ok fill:#dcfce7,stroke:#15803d,color:#052e16

    Q1{"Do you need to expose just ONE service<br/>(possibly overlapping CIDRs, one-way)?"}:::q
    Q2{"HTTP/gRPC service-to-service with<br/>auth policies across many accounts?"}:::q
    Q3{"How many VPCs need full<br/>network-level connectivity?"}:::q
    Q4{"Multi-Region / global WAN<br/>with policy-based segmentation?"}:::q
    PL["🔗 PrivateLink<br/>(NLB + endpoint service)"]:::ok
    LAT["🕸️ VPC Lattice"]:::ok
    PEER["🔀 VPC peering"]:::ok
    TGW["🛰️ Transit Gateway<br/>(+ TGW peering between Regions)"]:::ok
    CWAN["🌐 AWS Cloud WAN"]:::ok

    Q1 -->|"yes"| PL
    Q1 -->|"no"| Q2
    Q2 -->|"yes"| LAT
    Q2 -->|"no"| Q3
    Q3 -->|"2-5, few changes"| PEER
    Q3 -->|"many, or on-prem too"| Q4
    Q4 -->|"no, 1-3 Regions"| TGW
    Q4 -->|"yes"| CWAN
```

| Option | Use when | Key limits |
|---|---|---|
| **VPC peering** | A few VPCs, high volume, lowest cost | Non-transitive. No overlapping CIDRs. Routes needed **on both sides** |
| **Transit Gateway** | Many VPCs and/or on-prem, with segmentation | Per attachment-hour + per GB. Static routes in the VPCs |
| **PrivateLink** | Expose **one service** across accounts (overlap OK) | One-way. Provider needs an NLB/GWLB (Module 06) |
| **Site-to-Site VPN** | Fast, encrypted link to an office or DC | 2 tunnels. 1.25 Gbps per tunnel (5 Gbps on TGW). MTU ~1446 |
| **Direct Connect** | Steady high bandwidth, predictable latency | Weeks to provision. **Not encrypted by default** |
| **Client VPN** | People into the VPC | Managed OpenVPN. Needs authorization rules + routes |
| **VPC Lattice / Cloud WAN** | Service-to-service networking with auth / a global managed WAN | Newer, higher-level options |

---

## 2. VPC peering

```mermaid
%%{init: {"flowchart": {"wrappingWidth": 240}}}%%
flowchart LR
    classDef regional fill:#dbeafe,stroke:#1d4ed8,color:#0b1b3a
    classDef bad fill:#fee2e2,stroke:#b91c1c,color:#450a0a
    classDef gw fill:#ffedd5,stroke:#c2410c,color:#431407

    VA["☁️ VPC A"]:::regional
    VB["☁️ VPC B<br/>has IGW + NAT"]:::regional
    VC["☁️ VPC C"]:::regional
    NET(("🌐 Internet")):::bad
    VA <-->|"peering ✅"| VB
    VB <-->|"peering ✅"| VC
    VA -.-x|"❌ A to C via B: NOT allowed"| VC
    VB --> NET
    VA -.-x|"❌ A to internet via B's IGW/NAT"| NET
```

- 1:1, same or different account and Region. Traffic stays on the AWS backbone. No hourly charge.
- **No edge-to-edge routing:** A can't use B's IGW, NAT, VPN, Direct Connect, or gateway endpoints.
- Add routes in **both** VPCs' subnet route tables (`peer CIDR → pcx-…`) and allow the peer's CIDR (or, in the same Region, its SG) in your SGs.
- Full mesh grows as N×(N−1)/2. Past a handful of VPCs, switch to Transit Gateway.

## 3. Transit Gateway: a regional hub router with VRFs

| Concept | Meaning | Network analogy |
|---|---|---|
| **Attachment** | A VPC, VPN, Direct Connect gateway, TGW peering, or Connect (GRE/BGP) | Interface |
| **TGW route table** | A routing table inside the TGW | VRF |
| **Association** | Each attachment uses **exactly one** TGW route table for traffic **coming from** it | Interface → VRF binding |
| **Propagation** | An attachment **installs its routes** into one or more TGW route tables | Route leaking |

```mermaid
%%{init: {"flowchart": {"wrappingWidth": 240}}}%%
flowchart LR
    classDef regional fill:#dbeafe,stroke:#1d4ed8,color:#0b1b3a
    classDef rt fill:#fef9c3,stroke:#a16207,color:#422006
    classDef ext fill:#f1f5f9,stroke:#475569,color:#0f172a

    subgraph ATT["Attachments"]
        P1["☁️ prod-1 10.1/16"]:::regional
        P2["☁️ prod-2 10.2/16"]:::regional
        D1["☁️ dev 10.3/16"]:::regional
        SS["☁️ shared 10.255/16"]:::regional
        ONP["🔐 VPN / DX to on-prem 192.168/16"]:::ext
    end
    subgraph TGWRT["🛰️ TGW route tables (VRFs)"]
        RTP["📋 rt-prod<br/>10.1/16 → prod-1 (prop)<br/>10.2/16 → prod-2 (prop)<br/>10.255/16 → shared (prop)<br/>192.168/16 → vpn (prop)"]:::rt
        RTD["📋 rt-dev<br/>10.3/16 → dev (prop)<br/>10.255/16 → shared (prop)<br/>10.0.0.0/8 → blackhole (static)"]:::rt
        RTS["📋 rt-shared<br/>all VPC + on-prem routes (prop)"]:::rt
    end

    P1 ==>|"associated"| RTP
    P2 ==>|"associated"| RTP
    D1 ==>|"associated"| RTD
    SS ==>|"associated"| RTS
    ONP ==>|"associated"| RTS
```

- A VPC attachment uses **one subnet per AZ** (use small dedicated `/28`s). An AZ without an attachment subnet can't reach the TGW.
- TGW **never** updates VPC route tables. Add `10.0.0.0/8 → tgw-…` yourself.
- For A to reach B, **A's associated table** needs a route to B and **B's associated table** needs a route back.
- Enable **appliance mode** on the attachment of a central inspection VPC, so both directions of a flow use the same firewall.
- Share a TGW across accounts with AWS RAM. Peer TGWs across Regions (static routes only).

---

## 4. Hybrid connectivity

```mermaid
%%{init: {"flowchart": {"wrappingWidth": 240}}}%%
flowchart LR
    classDef ext fill:#f1f5f9,stroke:#475569,color:#0f172a
    classDef gw fill:#ffedd5,stroke:#c2410c,color:#431407
    classDef global fill:#f3e8ff,stroke:#7e22ce,color:#1f1147
    classDef regional fill:#dbeafe,stroke:#1d4ed8,color:#0b1b3a

    subgraph ONP["🏢 Your data center"]
        RTR["🧭 Edge router / firewall<br/>public IP 203.0.113.10, ASN 65000"]:::ext
    end
    CGW["📇 Customer gateway (CGW)<br/>AWS resource that DESCRIBES your device:<br/>public IP + BGP ASN (+ cert)"]:::gw
    VPN["🔐 Site-to-Site VPN connection<br/>2 IPsec tunnels"]:::gw
    DXLOC["🏬 Direct Connect location<br/>(colo facility, cross-connect)"]:::ext
    VIF["🏷️ Virtual interfaces (VIFs)<br/>802.1Q VLAN + BGP session each"]:::gw
    DXGW["🌐 Direct Connect gateway<br/>(GLOBAL)"]:::global
    VGW["🔒 Virtual private gateway (VGW)<br/>attached to ONE VPC, Amazon ASN"]:::gw
    TGW["🛰️ Transit Gateway<br/>(regional hub, many VPCs)"]:::gw
    VPC1["☁️ VPC"]:::regional
    VPCS["☁️☁️ Many VPCs"]:::regional

    RTR --- CGW
    CGW --> VPN
    VPN --> VGW
    VPN --> TGW
    RTR --> DXLOC --> VIF
    VIF -->|"private VIF"| VGW
    VIF -->|"private VIF"| DXGW
    VIF -->|"transit VIF"| DXGW
    DXGW --> VGW
    DXGW --> TGW
    VGW --> VPC1
    TGW --> VPCS
```

### Site-to-Site VPN
- A **customer gateway** resource describes your device (public IP + BGP ASN). AWS gives you **two tunnels** in different AZs: configure **both**, and prefer BGP.
- It terminates on a **VGW** (one VPC, which can propagate routes into VPC route tables) or a **Transit Gateway** (many VPCs, ECMP across tunnels, 5 Gbps tunnels, accelerated VPN).

### Direct Connect

```mermaid
%%{init: {"flowchart": {"wrappingWidth": 240}}}%%
flowchart LR
    classDef ext fill:#f1f5f9,stroke:#475569,color:#0f172a
    classDef gw fill:#ffedd5,stroke:#c2410c,color:#431407
    classDef global fill:#f3e8ff,stroke:#7e22ce,color:#1f1147
    classDef svc fill:#fef3c7,stroke:#b45309,color:#451a03
    classDef regional fill:#dbeafe,stroke:#1d4ed8,color:#0b1b3a

    subgraph CUST["🏢 Your network"]
        CR["🧭 Customer router<br/>ASN 65000"]:::ext
    end
    subgraph LOC["🏬 Direct Connect location (colo)"]
        CROUTER["Your router or partner cage"]:::ext
        XC["🔌 Cross-connect (fiber)<br/>LOA-CFA from AWS"]:::ext
        AWSR["AWS DX router (port 10G)"]:::gw
        CROUTER --- XC --- AWSR
    end
    CR ===|"your circuit / partner"| CROUTER

    AWSR -->|"VLAN 101 + BGP<br/>PRIVATE VIF"| VGW["🔒 VGW of one VPC<br/>(same Region)"]:::gw
    AWSR -->|"VLAN 102 + BGP<br/>PRIVATE VIF"| DXGW["🌐 DX gateway (global)"]:::global
    AWSR -->|"VLAN 103 + BGP<br/>TRANSIT VIF"| DXGW2["🌐 DX gateway (global)"]:::global
    AWSR -->|"VLAN 104 + BGP<br/>PUBLIC VIF (public IPs)"| PUB["🪣 AWS public endpoints<br/>S3, DynamoDB, public APIs, all Regions"]:::svc

    DXGW --> VGWA["🔒 VGW, VPC in us-east-1"]:::gw
    DXGW --> VGWB["🔒 VGW, VPC in eu-west-1"]:::gw
    DXGW2 --> TGW1["🛰️ TGW us-east-1 → many VPCs"]:::gw
    DXGW2 --> TGW2["🛰️ TGW ap-south-1 → many VPCs"]:::gw
```

| VIF | Goes to | Use |
|---|---|---|
| **Private VIF** | A VGW, or a Direct Connect gateway → VGWs in any Region | VPC private IPs |
| **Transit VIF** | Direct Connect gateway → Transit Gateways in any Region | Many VPCs |
| **Public VIF** | AWS public endpoints (public IPs) | S3/public APIs over DX |

- The **Direct Connect gateway is global**, but it doesn't route between its own associations (it's not a VPC-to-VPC path).
- DX is **unencrypted**: use **MACsec** or run **IPsec VPN over DX**. For resilience, use two locations, or DX plus a VPN backup.

### Route preference (AWS → on-prem)
1. **Longest prefix** wins, even across DX and VPN.
2. For the same prefix: static route in the VPC route table > propagated. Then **Direct Connect > VPN static > VPN BGP**.
3. Influence on-prem → AWS with your own BGP settings. Influence AWS → on-prem with AS-path prepending or the DX local-preference communities (`7224:7100` / `7200` / `7300`).

---

## 5. Hands-on: VPC peering

```bash
VPC2=$(aws ec2 create-vpc --cidr-block 10.1.0.0/16 --query Vpc.VpcId --output text); save VPC2
SUB2=$(aws ec2 create-subnet --vpc-id $VPC2 --cidr-block 10.1.0.0/24 --availability-zone $AZ1 --query Subnet.SubnetId --output text); save SUB2
SG_PEER=$(aws ec2 create-security-group --vpc-id $VPC2 --group-name lab-sg-peer --description peer --query GroupId --output text); save SG_PEER
aws ec2 authorize-security-group-ingress --group-id $SG_PEER --ip-permissions 'IpProtocol=icmp,FromPort=-1,ToPort=-1,IpRanges=[{CidrIp=10.0.0.0/16}]'
PEER_ID=$(aws ec2 run-instances --image-id $AMI_ID --instance-type t3.micro --subnet-id $SUB2 --security-group-ids $SG_PEER \
  --metadata-options HttpTokens=required --query 'Instances[0].InstanceId' --output text); save PEER_ID
aws ec2 wait instance-running --instance-ids $PEER_ID
PEER_IP=$(aws ec2 describe-instances --instance-ids $PEER_ID --query 'Reservations[0].Instances[0].PrivateIpAddress' --output text); save PEER_IP

PCX=$(aws ec2 create-vpc-peering-connection --vpc-id $VPC_ID --peer-vpc-id $VPC2 --query VpcPeeringConnection.VpcPeeringConnectionId --output text); save PCX
aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id $PCX >/dev/null

# Route on ONE side only: ping fails (no return route)
aws ec2 create-route --route-table-id $RT_PRIV --destination-cidr-block 10.1.0.0/16 --vpc-peering-connection-id $PCX
# Return route (vpc2's subnet uses vpc2's MAIN table): ping works
VPC2_MAIN_RT=$(aws ec2 describe-route-tables --filters Name=vpc-id,Values=$VPC2 Name=association.main,Values=true --query 'RouteTables[0].RouteTableId' --output text); save VPC2_MAIN_RT
aws ec2 create-route --route-table-id $VPC2_MAIN_RT --destination-cidr-block 10.0.0.0/16 --vpc-peering-connection-id $PCX

echo "peer IP: $PEER_IP"
aws ec2-instance-connect ssh --instance-id $APP_ID --connection-type eice
  ping -c 3 <peer IP>
  exit
```

Try the ping between the two `create-route` commands to watch it fail, then succeed.

---

## Check yourself

<details><summary>A peers with B, and B peers with C. Can A reach C?</summary>No. Peering is non-transitive.</details>
<details><summary>TGW association vs propagation?</summary>Association: the single table used for traffic from that attachment. Propagation: which tables learn the attachment's routes.</details>
<details><summary>DX and VPN both advertise 192.168.0.0/16. Which does AWS use?</summary>Direct Connect.</details>
<details><summary>Is Direct Connect encrypted?</summary>No. Use MACsec or a VPN over DX.</details>

---
**Previous:** [Module 06](06-private-access-and-dns.md) · **Next:** [Module 08 — Operate & Review](08-operate-and-review.md)
