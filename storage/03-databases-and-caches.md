# Module 03 — Databases & Caches: RDS/Aurora, ElastiCache, DynamoDB

← [All tutorials](../README.md) · **Storage & Databases** (short tutorial), module 3 of 3

---

## 1. RDS & Aurora: managed relational databases

```mermaid
%%{init: {"flowchart": {"wrappingWidth": 300}}}%%
flowchart TB
    classDef compute fill:#ccfbf1,stroke:#0f766e,color:#042f2e
    classDef db fill:#e0e7ff,stroke:#4338ca,color:#1e1b4b
    classDef global fill:#f3e8ff,stroke:#7e22ce,color:#1f1147
    classDef gw fill:#ffedd5,stroke:#c2410c,color:#431407

    APP["🧩 App (EC2 / ECS / Lambda)<br/>🛡️ sg-app"]:::compute
    PROXY["🔀 RDS Proxy (optional)<br/>connection pooling, faster failover"]:::gw
    WEP["🧭 Writer endpoint<br/>mycluster.cluster-xyz.rds.amazonaws.com"]:::global
    REP["🧭 Reader endpoint<br/>mycluster.cluster-ro-xyz…"]:::global
    subgraph AZA["🅰️ AZ-a · DB subnet"]
        W["🗄️ Aurora WRITER<br/>🛡️ sg-db: 5432 from sg-app"]:::db
    end
    subgraph AZB["🅱️ AZ-b · DB subnet"]
        R1["🗄️ Aurora READER<br/>(failover target)"]:::db
    end
    subgraph AZC["🅲 AZ-c · DB subnet"]
        R2["🗄️ Aurora READER"]:::db
    end
    VOL["💾 Aurora cluster volume: 6 copies across 3 AZs<br/>(shared by writer and readers)"]:::global

    APP --> PROXY
    PROXY --> WEP
    PROXY --> REP
    WEP --> W
    REP --> R1
    REP --> R2
    W ~~~ VOL
```

| Feature | RDS (MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, Db2) | Aurora (MySQL / PostgreSQL compatible) |
|---|---|---|
| HA | **Multi-AZ**: synchronous standby, automatic failover (DNS flips, ~1–2 minutes). Or a Multi-AZ *cluster* with 2 readable standbys | Readers in other AZs are failover targets (typically < 30 s). Shared storage |
| Read scaling | Up to 15 read replicas (async) | Up to 15 readers on the same storage. Low replica lag |
| Serverless | — | **Aurora Serverless v2** (scales in fine-grained ACUs) |
| Global | Cross-Region read replicas | **Aurora Global Database** (~1 s replication) |

**Always:** use a **DB subnet group** of private subnets in 2+ AZs; SG `sg-db` allows the port **from `sg-app` only**; store credentials in **Secrets Manager** (managed rotation) or use **IAM auth**; turn on automated backups (point-in-time restore), deletion protection, Performance Insights / Database Insights. Connect through **endpoints, never instance IPs**.

## 2. ElastiCache (and MemoryDB)

- Managed in-memory engines: **Valkey** (open-source Redis fork, AWS's recommended default), **Redis OSS**, **Memcached**. Choose **serverless** (no sizing) or node-based clusters.
- Valkey/Redis: **cluster mode** (sharding) + **replicas** + **Multi-AZ automatic failover**. Memcached: simple, multi-threaded, no replication.
- VPC-only: a subnet group plus SG port **6379** from the app SG. Enable in-transit TLS and AUTH/RBAC.
- Patterns: **cache-aside** (read the cache, on a miss read the DB and populate with a TTL), session store, rate limiting, leaderboards (sorted sets), pub/sub.
- Needs **durability** (cache as the primary database)? Use **MemoryDB** (Multi-AZ transaction log).

## 3. DynamoDB in one paragraph

A serverless key-value/document database with single-digit-millisecond latency at any scale. You design around **access patterns**: a **partition key** (+ optional sort key) and **GSIs** for other queries. Use **on-demand** capacity (pay per request) or provisioned capacity with auto scaling. **TTL**, **Streams** (change data capture to Lambda), **PITR** backups, and **global tables** (multi-Region active-active). Reach it privately via the free **gateway endpoint**.

---

## Check yourself

<details><summary>Ten containers in two AZs need the same files. EBS or EFS?</summary>EFS. EBS is one AZ, one instance.</details>
<details><summary>How do private instances reach S3 without NAT costs?</summary>An S3 gateway endpoint (free) on their route tables.</details>
<details><summary>RDS Multi-AZ vs a read replica?</summary>Multi-AZ is a synchronous standby for HA (not readable in the classic form). A read replica is asynchronous, readable, and used for read scaling.</details>
<details><summary>When should you use MemoryDB instead of ElastiCache?</summary>When the in-memory store is your primary database and must not lose data.</details>

---
**Previous:** [Module 02](02-ebs-efs-s3.md) · **Back to:** [All tutorials](../README.md)
