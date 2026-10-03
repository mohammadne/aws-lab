# Module 01 — Identities & Access

← [All tutorials](../README.md) · **IAM & User Management** (short tutorial), module 1 of 3

> Who can do what in AWS: identities (root, users, roles, SSO), policies, how AWS evaluates them, and the setup you should use.

*About a 20-minute read across 3 short modules, plus a 10-minute lab in Module 03.*

---

## 1. The mental model

Every AWS API call is **authenticated** (who are you?) and then **authorized** (are you allowed to do this to this resource?).

```mermaid
%%{init: {"flowchart": {"wrappingWidth": 300}}}%%
flowchart LR
    classDef ext fill:#f1f5f9,stroke:#475569,color:#0f172a
    classDef sec fill:#fee2e2,stroke:#b91c1c,color:#450a0a
    classDef global fill:#f3e8ff,stroke:#7e22ce,color:#1f1147
    classDef svc fill:#fef3c7,stroke:#b45309,color:#451a03

    subgraph WHO["Principals"]
        H["👩‍💻 Humans<br/>(via IAM Identity Center / SSO)"]:::ext
        A["⚙️ Workloads<br/>EC2, ECS tasks, Lambda, CI"]:::ext
        U["👤 IAM users<br/>(long-term keys: avoid)"]:::ext
    end
    STS["🎫 STS<br/>temporary credentials<br/>(key + secret + session token, expire)"]:::global
    EVAL["⚖️ Policy evaluation<br/>identity + resource policies,<br/>boundaries, SCPs/RCPs, session policies"]:::sec
    RES["🪣 Resource<br/>e.g. s3://bucket/key"]:::svc

    H -->|"assume a role via a permission set"| STS
    A -->|"assume an IAM role"| STS
    STS -->|"signed request (SigV4)"| EVAL
    U -->|"access keys (permanent)"| EVAL
    EVAL -->|"allow / deny"| RES
```

**Golden rule:** humans and workloads should use **roles with temporary credentials**. Long-lived access keys are the #1 cause of AWS account compromise.

## 2. Identities

| Identity | What it is | Use it for |
|---|---|---|
| **Root user** | The account's owner email. Unlimited power, can't be restricted by IAM | **Almost nothing.** Enable MFA, delete its access keys, lock it away (billing/account-level tasks only) |
| **IAM Identity Center** (SSO) users/groups | Workforce identities (built-in directory or your IdP: Entra ID, Okta, Google) | **All human access**, across all accounts in your AWS Organization |
| **Permission set** | A template of policies that Identity Center turns into a role in each assigned account | "Developers get PowerUser in dev, ReadOnly in prod" |
| **IAM role** | An identity **with no permanent credentials**. Whoever is trusted can assume it and gets temporary credentials | EC2 instance profiles, ECS task roles, Lambda, cross-account access, CI (OIDC) |
| **IAM user** | An identity with a password and/or **long-term access keys** | Only legacy tools or third parties that can't assume roles |
| **IAM group** | A set of IAM users. Attach policies to groups, not to users | Managing IAM users (if you must have them) |
| **Service-linked role** | A role owned by an AWS service (e.g. `AWSServiceRoleForECS`) | Created automatically, don't touch |

### Recommended human access setup

```mermaid
%%{init: {"flowchart": {"wrappingWidth": 300}}}%%
flowchart LR
    classDef ext fill:#f1f5f9,stroke:#475569,color:#0f172a
    classDef global fill:#f3e8ff,stroke:#7e22ce,color:#1f1147
    classDef sec fill:#fee2e2,stroke:#b91c1c,color:#450a0a
    classDef regional fill:#dbeafe,stroke:#1d4ed8,color:#0b1b3a

    IDP["🪪 Identity provider<br/>Entra ID / Okta / Google<br/>(or the Identity Center directory)"]:::ext
    IC["🔐 IAM Identity Center<br/>(org management account)"]:::global
    PS1["📜 Permission set: Developer"]:::sec
    PS2["📜 Permission set: ReadOnly"]:::sec
    DEV["☁️ dev account<br/>role AWSReservedSSO_Developer_…"]:::regional
    PROD["☁️ prod account<br/>role AWSReservedSSO_ReadOnly_…"]:::regional
    CLI["💻 aws configure sso / aws sso login<br/>browser login → short-lived creds"]:::ext

    IDP -->|"SAML/SCIM (users, groups)"| IC
    IC --> PS1 --> DEV
    IC --> PS2 --> PROD
    CLI --> IC
```

---
**Next:** [Module 02 — Policies & How AWS Evaluates Them](02-policies-and-evaluation.md)
