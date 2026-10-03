# AWS Tutorials

Compact, diagram-driven tutorials for working with AWS. Each one explains how the service works internally, gives decision tables for design choices, and has hands-on AWS CLI labs. Every tutorial is a folder of **ordered modules**: start at `01-…md` and follow the **Next** links.

**Suggested order:** IAM → Networking → Storage → ECS → Observability. The short tutorials can be read any time.

## 1. [Networking: VPC, EC2 & Beyond](networking/01-foundations.md) (8 modules)

VPCs, subnets, route tables, IGW/NAT, security groups & NACLs, EC2 networking, load balancers, endpoints, DNS, peering, Transit Gateway, VPN, Direct Connect.

| # | Module |
|---|---|
| 01 | [Foundations: how AWS networking works](networking/01-foundations.md) *(includes the tutorial overview)* |
| 02 | [VPC, subnets & route tables](networking/02-vpc-subnets-and-route-tables.md) |
| 03 | [Internet access: IGW, NAT, public IPs](networking/03-internet-access.md) |
| 04 | [Security groups & NACLs](networking/04-security-groups-and-nacls.md) |
| 05 | [EC2 networking & load balancers](networking/05-ec2-networking-and-load-balancers.md) |
| 06 | [Private access & DNS](networking/06-private-access-and-dns.md) |
| 07 | [Connecting networks](networking/07-connecting-networks.md) |
| 08 | [Operate & review](networking/08-operate-and-review.md) *(lab cleanup)* |

## 2. [Amazon ECS: Containers on Fargate, EC2 & Spot](ecs/01-ecs-fundamentals.md) (7 modules)

Task definitions, Fargate vs EC2 vs Managed Instances, Spot, capacity providers, awsvpc networking, ELB internals, Service Connect, deployments, auto scaling, security, troubleshooting.

| # | Module |
|---|---|
| 01 | [ECS fundamentals: object model & architecture](ecs/01-ecs-fundamentals.md) *(includes the tutorial overview)* |
| 02 | [Task definitions](ecs/02-task-definitions.md) |
| 03 | [Compute: Fargate, EC2, Spot & Managed Instances](ecs/03-compute-fargate-ec2-spot.md) |
| 04 | [Networking & load balancing](ecs/04-networking-and-load-balancing.md) |
| 05 | [Services, deployments & scaling](ecs/05-services-deployments-scaling.md) |
| 06 | [Security, observability & troubleshooting](ecs/06-security-observability-troubleshooting.md) |
| 07 | [Big picture & decisions](ecs/07-big-picture-and-decisions.md) *(lab cleanup)* |

## 3. [IAM & User Management](iam/01-identities-and-access.md) (short, 3 modules)

| # | Module |
|---|---|
| 01 | [Identities & access](iam/01-identities-and-access.md) |
| 02 | [Policies & how AWS evaluates them](iam/02-policies-and-evaluation.md) |
| 03 | [Roles, best practices & lab](iam/03-roles-best-practices-and-lab.md) |

## 4. [Storage & Databases](storage/01-choosing-storage.md) (short, 3 modules)

| # | Module |
|---|---|
| 01 | [Choosing the right store](storage/01-choosing-storage.md) |
| 02 | [Block, file & object: EBS, EFS, S3](storage/02-ebs-efs-s3.md) *(S3 lab)* |
| 03 | [Databases & caches: RDS/Aurora, ElastiCache, DynamoDB](storage/03-databases-and-caches.md) |

## 5. [Observability](observability/01-overview-and-metrics.md) (short, 3 modules)

| # | Module |
|---|---|
| 01 | [The observability map & CloudWatch metrics](observability/01-overview-and-metrics.md) |
| 02 | [Logs, alarms & lab](observability/02-logs-and-alarms.md) |
| 03 | [Traces & audit: X-Ray, CloudTrail, Config](observability/03-traces-and-audit.md) |

---

## Conventions

```text
<tutorial>/
├── 01-<topic>.md   # module 1 (full tutorials start with a short tutorial overview: course map, key questions)
├── 02-<topic>.md   # each module: concepts → diagrams → decision tables → gotchas → hands-on lab → "check yourself"
└── ...             # full tutorials: ≤ 8 modules · short tutorials: 3 modules
```

- **Labs** run in **bash**. Full tutorials save variables to an env file (`~/aws-lab.env` for networking, `~/ecs-lab.env` for ECS) so you can resume, and **the last module has the cleanup**. Short tutorials' mini labs clean up after themselves.
- **Diagrams** are Mermaid. They render on **GitHub/GitLab** (hover for zoom and full-screen), **Obsidian**, and **VS Code** with the *Markdown Preview Mermaid Support* extension. You can also paste any block into [mermaid.live](https://mermaid.live).

### Diagram legend (shared by all tutorials)

| Color | Meaning |
|---|---|
| 🟪 purple | Global / AWS-managed control plane |
| 🟦 blue | Regional resource / private subnet |
| 🟩 green | Zonal resource / public subnet |
| 🟥 red | Security (security groups, NACLs, IAM) |
| 🟧 orange | Gateways, load balancers, connections |
| 🟨 yellow | Route tables, configuration |
| 🩵 teal | Compute (instances, tasks, containers) |
| ⬜ gray | External (users, internet, on-prem) |

**Arrows:** 🔴 inbound from the internet · 🟢 egress via NAT · 🟣 private access to AWS services · 🔵 internal/VPC-to-VPC · 🟠 hybrid · dashed = association/configuration.

> [!WARNING]
> The labs create billable resources (NAT gateways, load balancers, Fargate tasks, public IPv4 addresses). Set an AWS budget alert, and always run each tutorial's cleanup.
