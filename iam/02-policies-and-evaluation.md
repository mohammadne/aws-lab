# Module 02 — Policies & How AWS Evaluates Them

← [All tutorials](../README.md) · **IAM & User Management** (short tutorial), module 2 of 3

---

## 1. Policies

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "ReadAppBucket",
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:ListBucket"],
    "Resource": ["arn:aws:s3:::my-app-bucket", "arn:aws:s3:::my-app-bucket/*"],
    "Condition": { "Bool": { "aws:SecureTransport": "true" } }
  }]
}
```

| Policy type | Attached to | Purpose |
|---|---|---|
| **Identity-based** (AWS-managed, customer-managed, inline) | Users, groups, roles | What this identity can do |
| **Resource-based** (S3 bucket, KMS key, SQS, Secrets Manager, Lambda…) | The resource; has a `Principal` | Who can access this resource, **including other accounts** |
| **Trust policy** | A role | **Who may assume** the role |
| **Permissions boundary** | A user or role | The *maximum* permissions it can ever get (for delegated admin) |
| **SCP / RCP** (AWS Organizations) | Accounts / OUs | Org-wide guardrails on principals (SCP) and resources (RCP). They never grant |
| **Session policy** | Passed at AssumeRole | Narrows one session |

### How AWS decides

```mermaid
%%{init: {"flowchart": {"wrappingWidth": 260}}}%%
flowchart TB
    classDef q fill:#fef9c3,stroke:#a16207,color:#422006
    classDef deny fill:#fee2e2,stroke:#b91c1c,color:#450a0a
    classDef ok fill:#dcfce7,stroke:#15803d,color:#052e16

    R(["📨 Request: principal + action + resource + context"]):::q
    D1["❓ Any explicit DENY in any applicable policy?"]:::q
    D2["❓ SCPs / RCPs allow it?"]:::q
    D3["❓ Resource policy allows it?<br/>(same account: enough on its own for many services)"]:::q
    D4["❓ Identity policy allows it?"]:::q
    D5["❓ Permissions boundary + session policy allow it?"]:::q
    DENY["⛔ DENY<br/>(default: everything is implicitly denied)"]:::deny
    ALLOW["✅ ALLOW"]:::ok

    R --> D1
    D1 -->|"yes"| DENY
    D1 -->|"no"| D2
    D2 -->|"no"| DENY
    D2 -->|"yes"| D3
    D3 -->|"yes"| ALLOW
    D3 -->|"no"| D4
    D4 -->|"no"| DENY
    D4 -->|"yes"| D5
    D5 -->|"no"| DENY
    D5 -->|"yes"| ALLOW
```

Key rules:
- **An explicit deny always wins.** Without any allow, the answer is deny.
- **Cross-account access** needs **both** sides: the caller's identity policy allows it, **and** the resource policy (or the role trust policy) in the other account allows it.
- **Within one account**, an allow in **either** the identity policy **or** the resource policy is usually enough, unless a deny, SCP/RCP, or boundary blocks it. (The chart above is simplified. See the AWS docs "Policy evaluation logic" for the edge cases.)

---
**Previous:** [Module 01](01-identities-and-access.md) · **Next:** [Module 03 — Roles, Best Practices & Lab](03-roles-best-practices-and-lab.md)
