# Module 01 — How Access to AWS Works

← [All tutorials](../README.md) · **IAM tutorial**, module 1 of 4

Everything you do in AWS is an **API request**. That includes clicking *Launch instance* in the web console, running `aws s3 ls` in a terminal, and your application uploading a file. For every request, AWS asks two questions:

1. **Who is asking?** This is *authentication*.
2. **Is this caller allowed to do this, to this thing, right now?** This is *authorization*.

**IAM (Identity and Access Management)** is the AWS service that answers both questions. It's global (not tied to a Region) and free. This tutorial teaches IAM by following real requests:

- **Module 01 (this one):** what happens when a person uses AWS, plus the identities IAM gives you: root user, IAM users, groups, roles, and IAM Identity Center.
- **Module 02:** how AWS services and your applications get permissions, without passwords or keys.
- **Module 03:** permission policies, read line by line, and how AWS decides allow or deny.
- **Module 04:** setting up an account the right way, with a hands-on lab.

---

## 1. Follow one request from start to finish

**Scenario.** Alice is a developer at a small company. She wants to see which files are in the S3 bucket `shop-uploads` (an **S3 bucket** is a named storage container for files).

### Step 1: Alice signs in

Her company uses **IAM Identity Center**, which gives employees a single sign-in page for AWS (Section 3.5 explains it). She configures the AWS CLI once:

```console
$ aws configure sso                    # (prompts abbreviated)
SSO start URL: https://mycompany.awsapps.com/start
SSO region: eu-central-1
# a browser opens; Alice logs in with her company account and approves
There are 2 AWS accounts available to you.
> dev  (111122223333)
  prod (444455556666)
There are 2 roles available to you.
> Developer
  ReadOnly
CLI profile name [Developer-111122223333]: dev
```

After that, her daily routine is a single command. It opens the browser, she logs in, and the CLI receives **temporary credentials** that expire after a few hours:

```console
$ aws sso login --profile dev
Successfully logged into Start URL: https://mycompany.awsapps.com/start
```

### Step 2: Alice runs a command

```console
$ aws s3 ls s3://shop-uploads/ --profile dev
```

The CLI turns this into an HTTPS request to the S3 API, an operation called `ListObjectsV2`. Before sending it, the CLI **signs** the request with Alice's secret key. The signature proves who sent it **without** sending the secret itself. It also expires after a few minutes, so a captured request can't be replayed later.

### Step 3: AWS authenticates the request

S3 checks the signature and works out the **principal**, meaning *who* is making the request:

```text
arn:aws:sts::111122223333:assumed-role/AWSReservedSSO_Developer_3f2a1b/alice@mycompany.com
```

In plain words: "In account `111122223333`, someone who took on the role `Developer` (as Alice) is calling." You can always ask AWS who it thinks you are:

```console
$ aws sts get-caller-identity --profile dev
{
    "UserId": "AROAEXAMPLEID:alice@mycompany.com",
    "Account": "111122223333",
    "Arn": "arn:aws:sts::111122223333:assumed-role/AWSReservedSSO_Developer_3f2a1b/alice@mycompany.com"
}
```

> `aws sts get-caller-identity` is the "whoami" of AWS. Run it first whenever permissions behave strangely. Half the time you're simply using a different identity or account than you think.

### Step 4: AWS builds the request context

S3 describes the request in IAM terms:

| Part | Value for Alice's request |
|---|---|
| **Principal** (who) | The `Developer` role, session `alice@mycompany.com` |
| **Action** (what) | `s3:ListBucket` |
| **Resource** (on what) | `arn:aws:s3:::shop-uploads` |
| **Context** (circumstances) | Time, source IP, whether MFA was used, the Region, tags on the role, HTTPS or not… |

Notice that the API operation is `ListObjectsV2` but the **IAM action** is `s3:ListBucket`. API names and permission names usually match, but not always. The *Service Authorization Reference* in the AWS docs lists the exact action for every API.

### Step 5: AWS authorizes the request

AWS collects every **policy** (a JSON document of permissions) that applies to this request: the policies attached to the `Developer` role, plus the **bucket policy** attached to `shop-uploads`, if there is one. It then decides. The rules are simple, and Module 03 shows them in full:

- Everything is **denied by default**.
- A matching **Allow** grants access.
- A matching **Deny** always wins over any Allow.

### Step 6: Alice gets an answer

If a policy allows `s3:ListBucket` on that bucket:

```console
2025-10-01 09:12:44      48213 uploads/banner.png
2025-10-01 09:13:02      91822 uploads/logo.png
```

If not:

```console
An error occurred (AccessDenied) when calling the ListObjectsV2 operation:
User: arn:aws:sts::111122223333:assumed-role/AWSReservedSSO_Developer_3f2a1b/alice@mycompany.com
is not authorized to perform: s3:ListBucket on resource: "arn:aws:s3:::shop-uploads"
because no identity-based policy allows the s3:ListBucket action
```

Read that error carefully. It tells you **who** was denied, **which action**, **on which resource**, and **why** ("no identity-based policy allows…"). That's exactly what you need to fix the policy.

**The web console works the same way.** Clicking around the console calls the same APIs, signed with the same kind of credentials. Every console error like "You don't have permission to …" is the same AccessDenied.

---

## 2. Vocabulary you'll use everywhere

| Term | Meaning | Example |
|---|---|---|
| **AWS account** | A container for resources and identities, identified by a 12-digit ID. Companies usually have several (dev, prod…) | `111122223333` |
| **Principal** | An identity that makes requests | A role, an IAM user, an AWS service |
| **Action** | A permission name, written `service:Operation` | `s3:GetObject`, `ec2:RunInstances` |
| **Resource** | The thing the action is performed on, identified by an ARN | A bucket, an instance, a role |
| **ARN** (Amazon Resource Name) | A globally unique name for any AWS resource | `arn:aws:s3:::shop-uploads` |
| **Policy** | A JSON document listing allowed or denied actions on resources | Module 03 |
| **Credentials** | What proves identity: a password (console), or an **access key ID + secret access key** (API), plus a **session token** if temporary | `AKIA…` (long-term), `ASIA…` (temporary) |

### Reading an ARN

```text
arn : aws : ec2 : eu-central-1 : 111122223333 : instance/i-0abc123def4567890
 │     │     │         │              │              └─ resource type / resource ID
 │     │     │         │              └─ account that owns it
 │     │     │         └─ Region (empty for global services like IAM and S3 buckets)
 │     │     └─ service
 │     └─ partition (aws, aws-cn, aws-us-gov)
 └─ always "arn"
```

| Resource | ARN | Why it looks like that |
|---|---|---|
| S3 bucket | `arn:aws:s3:::shop-uploads` | Bucket names are globally unique, so there's no Region or account |
| S3 object | `arn:aws:s3:::shop-uploads/uploads/logo.png` | The bucket ARN + `/` + object key |
| IAM role | `arn:aws:iam::111122223333:role/shop-app` | IAM is global, so there's no Region |
| EC2 instance | `arn:aws:ec2:eu-central-1:111122223333:instance/i-0abc…` | Regional and per account |

---

## 3. The identities IAM gives you

### 3.1 The root user

When an AWS account is created, it comes with a **root user**: the email address and password used to sign up. Root can do **everything**, including closing the account and changing billing, and **IAM policies can't restrict it**.

What to do with it:
1. Turn on **MFA** (multi-factor authentication: a phone app or hardware key) right away.
2. **Never create access keys** for root.
3. Use root only for the few tasks that require it (changing account settings, closing the account, restoring access if all admins are locked out). Everything else uses the identities below.

### 3.2 IAM users

An **IAM user** is a named identity inside **one** account, with **long-term credentials**:
- a **console password** (to sign in to the web console), and/or
- **access keys** (an access key ID `AKIA…` plus a secret) for the CLI and SDKs.

```console
$ aws iam create-user --user-name bob
$ aws iam create-login-profile --user-name bob --password '…' --password-reset-required   # console access
$ aws iam create-access-key --user-name bob                                              # API access (avoid!)
```

**Why IAM users are discouraged for people today:** their access keys never expire on their own. They get copied into laptops, CI settings, and accidentally into git repositories, where attackers find them within minutes. Each user also exists in only one account, so ten accounts would mean ten users for Bob. Use IAM users only for software that genuinely can't use roles (some older third-party tools). Even then, rotate the keys and restrict them tightly.

### 3.3 IAM groups

A **group** is a collection of IAM users. You attach permissions to the group, and every member gets them: `developers`, `auditors`, `admins`. A user can be in several groups. Groups are only for organizing users, so you can't sign in as a group or reference a group as a principal in a policy.

### 3.4 IAM roles

A **role** is an identity with permissions but **no password and no long-term keys**. Instead, a trusted principal **assumes** the role and receives **temporary credentials** (access key `ASIA…`, secret, and **session token**) that expire after 15 minutes to 12 hours.

Think of a role as a **badge** hanging on a hook. The badge says what doors it opens (**permission policies**), and a note next to it says who may pick it up (**trust policy**). Whoever picks it up acts with the badge's permissions until it expires.

Roles are the backbone of modern AWS access:
- **People** get roles through Identity Center (Alice's `Developer` role above).
- **AWS services and your apps** get roles (Module 02).
- **Other accounts** get roles to work across accounts.
- **CI/CD systems** like GitHub Actions get roles without storing any keys.

### 3.5 IAM Identity Center: how people should sign in

**IAM Identity Center** (formerly *AWS SSO*) manages **people** for **all** your AWS accounts in one place:

1. **Users and groups** come from its built-in directory, or are synced from your company directory (Microsoft Entra ID, Google Workspace, Okta…).
2. You define **permission sets**: named bundles of permissions such as `Developer`, `ReadOnly`, `Admin`.
3. You **assign** a group + permission set to an account: "group `backend-team` gets `Developer` in `dev` and `ReadOnly` in `prod`".
4. Identity Center creates a matching **role** in each assigned account (named `AWSReservedSSO_Developer_…`). When a user signs in to the **access portal** and picks an account and permission set, they're assuming that role.

That's exactly what Alice did in Section 1. People get one login, MFA in one place, and access that disappears from every account when they leave the company.

### 3.6 Which identity for what

| Identity | Credentials | Expire? | Use it for |
|---|---|---|---|
| Root user | Email + password (+ MFA) | No | Almost nothing: account-level tasks only |
| Identity Center user | Company login → temporary role credentials | **Yes** (hours) | **All people** |
| IAM user | Password and/or access keys | **No** | Legacy software that can't use roles |
| IAM group | — (organizes IAM users) | — | Giving several IAM users the same permissions |
| IAM role | Temporary credentials when assumed | **Yes** | Applications, AWS services, cross-account access, CI/CD, people via SSO |

---

## 4. Try it (safe, read-only plus a throwaway user)

You need the AWS CLI v2 and credentials with IAM read/write permissions (an administrator profile).

```bash
# Who am I?
aws sts get-caller-identity

# What identities exist in this account?
aws iam list-users  --query 'Users[].UserName'
aws iam list-roles  --query 'Roles[].RoleName' --output text | tr '\t' '\n' | head -20
#   Names starting with "AWSServiceRoleFor…" are created by AWS services (Module 02).
#   Names starting with "AWSReservedSSO_…" come from Identity Center.

# Create a group with read-only permissions, and a user with NO credentials in it
aws iam create-group --group-name lab-readers
aws iam attach-group-policy --group-name lab-readers --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess
aws iam create-user --user-name lab-alice
aws iam add-user-to-group --user-name lab-alice --group-name lab-readers

# Ask IAM what lab-alice would be allowed to do, without her ever signing in
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::$ACCOUNT_ID:user/lab-alice \
  --action-names s3:ListBucket ec2:DescribeInstances ec2:TerminateInstances iam:CreateUser \
  --query 'EvaluationResults[].[EvalActionName,EvalDecision]' --output table
```

Expected result:

```text
|  s3:ListBucket          |  allowed       |
|  ec2:DescribeInstances  |  allowed       |
|  ec2:TerminateInstances |  implicitDeny  |   <- nothing allows it, so it's denied by default
|  iam:CreateUser         |  implicitDeny  |
```

`implicitDeny` means "no policy allowed it". `explicitDeny` would mean "a policy explicitly denied it". Clean up:

```bash
aws iam remove-user-from-group --user-name lab-alice --group-name lab-readers
aws iam delete-user --user-name lab-alice
aws iam detach-group-policy --group-name lab-readers --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess
aws iam delete-group --group-name lab-readers
```

---

## Check yourself

<details><summary>What does AWS determine in "authentication", and what in "authorization"?</summary>Authentication: who sent the request (verified by the request signature). Authorization: whether the policies allow this principal to perform this action on this resource in this context.</details>
<details><summary>A colleague asks for an IAM user with access keys for daily CLI work. What do you set up instead?</summary>Identity Center access with a permission set. They run `aws sso login` and get temporary credentials that expire.</details>
<details><summary>What's the difference between an IAM user and a role?</summary>A user has long-term credentials and belongs to one person or app. A role has no long-term credentials. Trusted principals assume it and receive temporary ones.</details>
<details><summary>`aws s3 ls s3://bucket` fails with AccessDenied for s3:ListBucket. What's the first command you run?</summary>`aws sts get-caller-identity`, to confirm which identity and account you're actually using.</details>

---
**Next:** [Module 02 — How AWS Services Use IAM](02-how-services-use-iam.md)
