# Module 04 — Setting Up an Account the Right Way

← [All tutorials](../README.md) · **IAM tutorial**, module 4 of 4

You now know how requests are authorized (Module 01), how software gets credentials (Module 02), and how policies work (Module 03). This module puts it together: the setup a real team should use, step by step, and how to **audit** an existing account.

---

## 1. Step 1: lock away the root user

In the console, signed in as root (top-right menu → *Security credentials*):

1. **Assign MFA**: an authenticator app or a hardware security key.
2. **Delete any access keys** listed for root.
3. Set **alternate contacts** (billing, operations, security) so AWS can reach the right people.
4. Sign out. From now on, root is for emergencies only.

## 2. Step 2: give people access through IAM Identity Center

1. Open **AWS Organizations** and create an organization. The account you're in becomes the **management account**. Keep workloads out of it. Create or invite member accounts such as `dev` and `prod`.
2. Open **IAM Identity Center** → *Enable*.
3. **Identity source**: keep the built-in directory, or connect your company IdP (Microsoft Entra ID, Google Workspace, Okta) so users and groups sync automatically.
4. Create **groups** that match how people work: `admins`, `backend`, `data`, `auditors`.
5. Create **permission sets**: start with AWS managed ones (`AdministratorAccess`, `PowerUserAccess`, `ReadOnlyAccess`), then add custom ones as needs become clear. Set the **session duration** (for example 8 hours).
6. **Assign**: `backend` → `PowerUserAccess` in `dev`, `ReadOnlyAccess` in `prod`. `admins` → `AdministratorAccess` everywhere.
7. Everyone uses the **access portal** URL for the console, and `aws configure sso` / `aws sso login` for the CLI (Module 01).

When someone leaves, disable them in one place and all their AWS access is gone.

## 3. Step 3: one role per application

For every workload, create **its own role** with **only** what it needs, using a customer-managed policy (Modules 02–03). No application should ever have access keys. Name roles after the app and environment (`shop-api-prod`), so CloudTrail logs are readable.

## 4. Step 4: CI/CD without stored keys (GitHub Actions example)

Instead of saving access keys as repository secrets, let GitHub prove its identity with an **OIDC token** (a signed statement from GitHub saying "this job runs in repo `myorg/shop` on branch `main`") and exchange it for a role.

**4.1 Register GitHub as an identity provider.** IAM console → *Identity providers* → *Add provider* → *OpenID Connect*:
- Provider URL: `https://token.actions.githubusercontent.com`
- Audience: `sts.amazonaws.com`

**4.2 Create a role with this trust policy:**

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Federated": "arn:aws:iam::111122223333:oidc-provider/token.actions.githubusercontent.com" },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
        "token.actions.githubusercontent.com:sub": "repo:myorg/shop:ref:refs/heads/main"
      }
    }
  }]
}
```

- **`Federated`**: trust tokens from the GitHub provider registered in 4.1.
- **`AssumeRoleWithWebIdentity`**: the STS action that swaps an OIDC token for AWS credentials.
- **`sub`** is the most important line. Only workflows from repository `myorg/shop` on branch `main` may assume the role. **Without it, any GitHub repository in the world could.**

**4.3 Use it in the workflow:**

```yaml
permissions:
  id-token: write        # allow the job to request an OIDC token
  contents: read
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::111122223333:role/github-deploy-shop
          aws-region: eu-central-1
      - run: aws sts get-caller-identity     # assumed-role/github-deploy-shop/GitHubActions
```

The same pattern works for GitLab CI, Bitbucket, CircleCI, and Terraform Cloud.

## 5. Step 5: guardrails with SCPs

A **Service Control Policy** (Module 03 §4), attached to the `Workloads` organizational unit, can enforce rules that even account admins can't break. Example: only allow EU Regions.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyNonEURegions",
    "Effect": "Deny",
    "NotAction": [
      "iam:*", "organizations:*", "sts:*", "support:*", "budgets:*",
      "cloudfront:*", "route53:*", "waf:*", "globalaccelerator:*"
    ],
    "Resource": "*",
    "Condition": {
      "StringNotEquals": { "aws:RequestedRegion": ["eu-central-1", "eu-west-1"] }
    }
  }]
}
```

- **`Effect: Deny` + `NotAction`**: deny **every** action **except** the listed global services. Global services always report `us-east-1`, so blocking them would break the account. This is the one place where `NotAction` is the right tool.
- **`StringNotEquals` on `aws:RequestedRegion`**: the deny applies whenever the request targets a Region outside the list.

Other common SCPs: deny `organizations:LeaveOrganization`, deny stopping or deleting CloudTrail, deny creating IAM users or access keys.

## 6. Step 6: scale permissions with tags (ABAC)

Instead of writing one policy per team, write **one** policy that compares tags. **ABAC** stands for attribute-based access control. "You may start or stop instances whose `team` tag equals **your** `team` tag":

```json
{
  "Effect": "Allow",
  "Action": ["ec2:StartInstances", "ec2:StopInstances"],
  "Resource": "arn:aws:ec2:*:*:instance/*",
  "Condition": { "StringEquals": { "aws:ResourceTag/team": "${aws:PrincipalTag/team}" } }
}
```

Identity Center can pass user attributes (like department) as principal tags, so a new team needs **no new policy**, only tags.

## 7. Step 7: find and fix problems continuously

| Tool | What it tells you |
|---|---|
| **IAM Access Analyzer: external access** | Which resources (buckets, roles, keys, queues…) are shared **outside** your account or organization |
| **IAM Access Analyzer: unused access** | Roles, users, keys, and permissions that **haven't been used** (paid feature) |
| **Access Analyzer policy validation / generation** | Errors and risky patterns in a policy. Can **generate** a least-privilege policy from CloudTrail activity |
| **Last accessed information** | Per role or user, which services were used and when. Remove what's never used |
| **Credential report** | Every IAM user: password and MFA status, access key age and last use |
| **CloudTrail** | Who did what, when, from where. See the [Observability tutorial](../observability/03-traces-and-audit.md) |

### Final checklist
- [ ] Root: MFA, no access keys, contacts set
- [ ] People: Identity Center + MFA. **No IAM users for humans**
- [ ] Workloads: one role each, least privilege, **zero long-lived keys**
- [ ] CI/CD: OIDC roles restricted by repository and branch
- [ ] `iam:PassRole` granted narrowly
- [ ] Guardrails: SCPs (Regions, CloudTrail protection), permissions boundaries for teams that create roles
- [ ] Access Analyzer enabled. Unused access and old keys reviewed regularly

---

## 8. Try it: audit your own account (read-only)

```bash
# Root user health: MFA on (1)? root access keys present (should be 0)?
aws iam get-account-summary \
  --query 'SummaryMap.{RootMFA:AccountMFAEnabled,RootAccessKeys:AccountAccessKeysPresent,IAMUsers:Users,Roles:Roles,Policies:Policies}'

# Credential report: every IAM user, MFA status, access key age and last use
aws iam generate-credential-report >/dev/null; sleep 5
aws iam get-credential-report --query Content --output text | base64 --decode \
  | cut -d, -f1,4,8,9,10,11 | column -t -s,
#   columns: user, password_enabled, mfa_active, key1_active, key1_last_rotated, key1_last_used

# Which services has a role actually used? (pick any role name from `aws iam list-roles`)
ROLE_ARN=$(aws iam list-roles --query 'Roles[0].Arn' --output text)
JOB=$(aws iam generate-service-last-accessed-details --arn $ROLE_ARN --query JobId --output text); sleep 5
aws iam get-service-last-accessed-details --job-id $JOB \
  --query 'ServicesLastAccessed[?TotalAuthenticatedEntities>`0`].[ServiceNamespace,LastAuthenticated]' --output table

# What is shared outside the account? (an account analyzer is free)
aws accessanalyzer create-analyzer --analyzer-name lab-analyzer --type ACCOUNT >/dev/null 2>&1
sleep 30
ANALYZER=$(aws accessanalyzer list-analyzers --query "analyzers[?name=='lab-analyzer'].arn" --output text)
aws accessanalyzer list-findings --analyzer-arn $ANALYZER \
  --query 'findings[].[resourceType,resource,status]' --output table
# keep the analyzer (recommended), or remove it:
# aws accessanalyzer delete-analyzer --analyzer-name lab-analyzer
```

What to act on: root keys present, users without MFA, access keys older than 90 days or never used, roles with services never used, and any finding showing a resource shared with an account you don't recognize.

---

## Check yourself

<details><summary>What's the single most important condition in a GitHub OIDC trust policy?</summary>`token.actions.githubusercontent.com:sub`, restricting which repository (and branch) may assume the role.</details>
<details><summary>Why does the Region-restricting SCP use NotAction?</summary>To deny everything except global services (IAM, STS, Organizations, CloudFront…), which would otherwise be blocked because they're served from us-east-1.</details>
<details><summary>How do you find permissions a role never uses?</summary>Last accessed information (per role), or Access Analyzer's unused access findings, or policy generation from CloudTrail activity.</details>

---
**Previous:** [Module 03](03-policies-in-detail.md) · **Next tutorial:** [Networking](../networking/01-regions-azs-and-vpcs.md)
