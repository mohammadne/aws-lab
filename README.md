# AWS Tutorials

Practical tutorials for working with AWS. Each one introduces the concepts step by step, shows real commands and their output, explains what happened, and ends every module with a hands-on lab you can run with the AWS CLI.

Start each tutorial at its first module and follow the **Next** links. Suggested order:

### 1. [IAM: Users, Roles & Permissions](iam/01-how-access-works.md) (4 modules)
How AWS decides who may do what. Follow a request from sign-in to "allowed"/"AccessDenied", see how services and apps get credentials without keys, and read a policy line by line.
1. [How access to AWS works](iam/01-how-access-works.md)
2. [How AWS services use IAM](iam/02-how-services-use-iam.md)
3. [Policies in detail](iam/03-policies-in-detail.md)
4. [Setting up an account the right way](iam/04-secure-account-setup.md)

### 2. [Networking: VPC, EC2 & Load Balancers](networking/01-regions-azs-and-vpcs.md) (8 modules)
Build the network for a web application: VPC, subnets, routing, internet and NAT gateways, firewalls, servers, a load balancer, private access to AWS services, connections to other networks.
1. [Regions, Availability Zones & your first VPC](networking/01-regions-azs-and-vpcs.md)
2. [Subnets, routing & internet access](networking/02-subnets-routing-and-internet-access.md)
3. [Security groups & network ACLs](networking/03-security-groups-and-nacls.md)
4. [EC2 instances in your VPC](networking/04-ec2-instances-in-your-vpc.md)
5. [Load balancers](networking/05-load-balancers.md)
6. [Private access to AWS services & DNS](networking/06-private-access-and-dns.md)
7. [Connecting networks: peering, Transit Gateway, VPN, Direct Connect](networking/07-connecting-networks.md)
8. [Troubleshooting, costs, the full picture & cleanup](networking/08-troubleshooting-costs-and-cleanup.md)

### 3. [Storage & Databases](storage/01-storage-basics-and-s3.md) (3 modules)
Which store for which data, and how to use it: S3, EBS, EFS, RDS/Aurora, ElastiCache, DynamoDB.
1. [Storage basics & Amazon S3](storage/01-storage-basics-and-s3.md)
2. [Disks for servers: EBS & EFS](storage/02-ebs-and-efs.md)
3. [Databases & caches](storage/03-databases-and-caches.md)

### 4. [Amazon ECS: Running Containers](ecs/01-what-ecs-is-and-your-first-task.md) (6 modules)
Run containers in production: task definitions field by field, services behind a load balancer, Fargate vs EC2 vs Spot, deployments and rollbacks, scaling, debugging.
1. [What ECS is & your first task](ecs/01-what-ecs-is-and-your-first-task.md)
2. [The task definition, field by field](ecs/02-task-definition-field-by-field.md)
3. [Services, networking & load balancing](ecs/03-services-networking-and-load-balancing.md)
4. [Compute: Fargate, Fargate Spot, EC2 & Managed Instances](ecs/04-compute-fargate-ec2-spot.md)
5. [Deployments & scaling](ecs/05-deployments-and-scaling.md)
6. [Operating ECS: security, debugging, costs & the full picture](ecs/06-operating-ecs.md)

### 5. [Observability: CloudWatch, X-Ray & CloudTrail](observability/01-metrics.md) (3 modules)
See what your systems are doing: metrics, logs, alarms, traces, and who changed what.
1. [Observability basics & CloudWatch metrics](observability/01-metrics.md)
2. [Logs & alarms](observability/02-logs-and-alarms.md)
3. [Traces & audit](observability/03-traces-and-audit.md)

---

## Before you run the labs

- Install the **AWS CLI v2** and sign in with an administrator identity (the IAM tutorial shows the right way: `aws configure sso`).
- Run the commands in **bash**. The longer tutorials save resource IDs to a file (`~/aws-lab.env`, `~/ecs-lab.env`), so you can resume later.
- Some resources cost money while they exist: NAT gateways, load balancers, Fargate tasks, public IPv4 addresses. **Set a budget alert**, and run each tutorial's cleanup (the last module of Networking and ECS; the short labs clean up after themselves).

## Reading the diagrams

The diagrams are written in Mermaid. They render on GitHub and GitLab (hover over a diagram to zoom), in Obsidian, and in VS Code with the *Markdown Preview Mermaid Support* extension. Colors are used consistently:

🟪 global or AWS-managed · 🟦 regional / private subnet · 🟩 zonal / public subnet · 🟥 security · 🟧 gateways and load balancers · 🟨 route tables and configuration · 🩵 servers and containers · ⬜ outside AWS

Arrows: 🔴 traffic from the internet · 🟢 outbound through NAT · 🟣 private access to AWS services · 🔵 between VPCs or services · 🟠 to on-premises · dashed lines = an association, not traffic.
