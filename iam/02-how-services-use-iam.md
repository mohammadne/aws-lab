# Module 02 — How AWS Services Use IAM

← [All tutorials](../README.md) · **IAM tutorial**, module 2 of 4

In Module 01, a **person** called AWS. Most AWS calls, though, come from **software**: your web app uploading images to S3, a Lambda function writing to a database, AWS itself launching instances for an Auto Scaling group. Software needs credentials too. This module shows how it gets them **without anyone storing a password or access key**.

---

## 1. Walkthrough: an app on EC2 reads from S3

**Scenario.** The `shop` web app runs on an EC2 instance (a virtual server). It must read product images from the bucket `shop-uploads`.

### The wrong way: access keys in a config file

```ini
# /etc/shop/config.ini   <- don't do this
aws_access_key_id = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

These keys never expire, so anyone who copies the file, a backup, or the AMI image has permanent access. You can't tell which server used them, and rotating them means touching every server.

### The right way: give the instance a role

You create a role for the app. A role has two parts (Module 01 §3.4):

**1. Trust policy: who may assume the role.** Here, the EC2 service:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "ec2.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
```

**2. Permission policy: what the role can do.** Here, read objects from one bucket:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::shop-uploads/*"
  }]
}
```

(Module 03 explains every field of these documents.)

Then you **attach the role to the instance**. EC2 needs one small extra object for this, an **instance profile**. It's a container that holds one role so EC2 can hand it to an instance. The console creates it automatically, with the same name as the role. With the CLI:

```bash
aws iam create-role --role-name shop-app --assume-role-policy-document file://trust-ec2.json
aws iam put-role-policy --role-name shop-app --policy-name read-uploads --policy-document file://read-uploads.json
aws iam create-instance-profile --instance-profile-name shop-app
aws iam add-role-to-instance-profile --instance-profile-name shop-app --role-name shop-app
aws ec2 associate-iam-instance-profile --instance-id i-0abc123def4567890 --iam-instance-profile Name=shop-app
```

### What happens at runtime

1. EC2 assumes the role on the instance's behalf and makes **temporary credentials** available **inside the instance**, through the **instance metadata service (IMDS)**. IMDS is a local HTTP endpoint, `http://169.254.169.254`, reachable only from the instance itself.
2. Your code uses an AWS SDK (boto3, the Java SDK, the AWS CLI…) **without any credential configuration**. The SDK searches a fixed list of places, the **credential provider chain**:
   1. Environment variables (`AWS_ACCESS_KEY_ID`, …)
   2. Shared files (`~/.aws/credentials`, `~/.aws/config`)
   3. Container credentials (ECS, see the table below)
   4. **Instance metadata (IMDS)**: found here on EC2
3. The SDK **refreshes the credentials automatically** before they expire. Nothing to rotate, nothing to leak for long.

You can see it from a shell on the instance:

```console
$ aws sts get-caller-identity
{
    "Account": "111122223333",
    "Arn": "arn:aws:sts::111122223333:assumed-role/shop-app/i-0abc123def4567890"
}

$ TOKEN=$(curl -sX PUT http://169.254.169.254/latest/api/token -H "X-aws-ec2-metadata-token-ttl-seconds: 60")
$ curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/iam/security-credentials/shop-app
{
  "AccessKeyId" : "ASIA…",
  "SecretAccessKey" : "…",
  "Token" : "…",
  "Expiration" : "2025-10-01T15:42:07Z"
}
```

The session name is the **instance ID**, so CloudTrail logs show exactly which server did what. The `PUT …/api/token` step is **IMDSv2**, a session-token handshake that protects the credentials from certain web attacks (SSRF). Configure instances to **require IMDSv2**.

---

## 2. The same pattern everywhere

Every AWS compute service works this way: **you create a role that trusts that service, attach it to your workload, and the SDK finds the credentials automatically.** Only the names differ.

| Where your code runs | What you attach | Trust policy `Principal` | How the SDK gets credentials |
|---|---|---|---|
| **EC2** instance | Instance profile (holding a role) | `"Service": "ec2.amazonaws.com"` | Instance metadata (IMDS) |
| **ECS** task (container) | **Task role** | `"Service": "ecs-tasks.amazonaws.com"` | Container credentials endpoint (env var `AWS_CONTAINER_CREDENTIALS_RELATIVE_URI`) |
| **Lambda** function | **Execution role** | `"Service": "lambda.amazonaws.com"` | Environment variables injected by Lambda |
| **EKS** pod | EKS Pod Identity / IRSA | `pods.eks.amazonaws.com` / your cluster's OIDC provider | Token file + `AssumeRoleWithWebIdentity` |
| **GitHub Actions**, GitLab CI | A role trusted via **OIDC federation** | `"Federated": "…:oidc-provider/token.actions.githubusercontent.com"` | The CI job exchanges its signed OIDC token for credentials |
| Your laptop (a person) | Identity Center permission set | (managed by Identity Center) | `aws sso login` → `~/.aws/sso/cache` |

> **ECS has two roles.** The **task role** is for *your code* (reading S3, say). The **task execution role** is used by *ECS itself before your code starts*, to pull the container image and fetch secrets. The ECS tutorial covers this in detail.

---

## 3. Roles that AWS services use for themselves

Sometimes the **service itself** needs to act in your account: Auto Scaling launching instances, ECS registering tasks with a load balancer, CloudFormation creating resources.

| Kind | Who creates it | Can you edit its permissions? | Example |
|---|---|---|---|
| **Service role** | You (the console often offers to create it) | Yes | CloudFormation service role, ECS task execution role, CodeBuild role |
| **Service-linked role** | **The service, automatically**, the first time you use a feature | **No.** AWS defines and maintains it | `AWSServiceRoleForAutoScaling`, `AWSServiceRoleForECS`, `AWSServiceRoleForElasticLoadBalancing` |

You'll see service-linked roles in `aws iam list-roles` (names start with `AWSServiceRoleFor`). Leave them alone.

### `iam:PassRole`: who may hand a role to a service

To attach a role to an EC2 instance, an ECS task, or a Lambda function, **you** need the permission `iam:PassRole` on that role. This prevents privilege escalation. Without it, someone with only "launch instances" permission could attach the `Admin` role to an instance, log in, and become admin. Grant `iam:PassRole` narrowly:

```json
{
  "Effect": "Allow",
  "Action": "iam:PassRole",
  "Resource": "arn:aws:iam::111122223333:role/shop-*",
  "Condition": { "StringEquals": { "iam:PassedToService": "ec2.amazonaws.com" } }
}
```

---

## 4. The other direction: resource-based policies

So far, permissions were attached to the **identity** ("the role `shop-app` may read the bucket"). Many services also let you attach a policy to the **resource** itself, which says **who** may use it. This is a **resource-based policy**, and it has a `Principal` field naming who is allowed:

| Resource | Its policy is called | Typical use |
|---|---|---|
| S3 bucket | **Bucket policy** | Let another account read the bucket, require HTTPS, allow CloudFront |
| KMS key | **Key policy** (mandatory on every key) | Decide who can encrypt and decrypt with the key |
| SQS queue / SNS topic | Queue / topic policy | Let S3 or EventBridge send messages |
| Lambda function | Function policy | Let S3, API Gateway, or EventBridge **invoke** the function |
| Secrets Manager secret | Resource policy | Share a secret with another account |
| IAM role | **Trust policy** | Who may assume the role. It's a resource-based policy too |

**Example.** You configure "when a file lands in `shop-uploads`, invoke the Lambda function `make-thumbnail`". S3 itself calls Lambda, so the **function's** resource policy must allow the S3 service. The console adds this for you. Without the console, you'd run:

```bash
aws lambda add-permission --function-name make-thumbnail --statement-id s3-invoke \
  --action lambda:InvokeFunction --principal s3.amazonaws.com \
  --source-arn arn:aws:s3:::shop-uploads --source-account 111122223333
```

The `--source-arn` and `--source-account` conditions matter. Without them, **any** bucket in any account could trigger your function. This is called the *confused deputy* problem.

---

## 5. Working across accounts

Companies split environments into accounts (`dev`, `prod`, `shared`). To work across them, you **assume a role in the other account**:

1. In **prod** (`444455556666`), create a role `deployer` whose trust policy allows the dev account: `"Principal": { "AWS": "arn:aws:iam::111122223333:root" }`. (`:root` here means "the account 111122223333", not its root user. That account's admins then decide which of their identities may assume it.)
2. In **dev**, give the CI role permission `sts:AssumeRole` on `arn:aws:iam::444455556666:role/deployer`.
3. Use it. The CLI can assume the role automatically through a profile:

```ini
# ~/.aws/config
[profile prod-deployer]
role_arn       = arn:aws:iam::444455556666:role/deployer
source_profile = dev
```

```bash
aws s3 ls --profile prod-deployer      # the CLI calls sts:AssumeRole for you
```

For **third parties** (a monitoring vendor, say), the trust policy also requires an `sts:ExternalId` condition: a secret value agreed with that vendor.

---

## 6. Try it: create a role and use it like an application would

This role trusts **your own account** (standing in for a service), so you can assume it from your terminal.

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

cat > /tmp/trust.json <<EOF
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"AWS":"arn:aws:iam::$ACCOUNT_ID:root"},"Action":"sts:AssumeRole"}]}
EOF
aws iam create-role --role-name lab-app --assume-role-policy-document file:///tmp/trust.json >/dev/null
aws iam attach-role-policy --role-name lab-app --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
sleep 10   # new roles take a few seconds to become usable everywhere

# Add a profile that assumes the role, exactly like the cross-account example
aws configure set profile.lab-app.role_arn arn:aws:iam::$ACCOUNT_ID:role/lab-app
aws configure set profile.lab-app.source_profile ${AWS_PROFILE:-default}

aws sts get-caller-identity --profile lab-app           # .../assumed-role/lab-app/botocore-session-…
aws s3 ls --profile lab-app | head -3                   # allowed (S3 read-only)
aws ec2 describe-vpcs --profile lab-app 2>&1 | tail -1  # UnauthorizedOperation: nothing allows EC2
```

Clean up:

```bash
aws iam detach-role-policy --role-name lab-app --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
aws iam delete-role --role-name lab-app
rm -f /tmp/trust.json
# finally, delete the 3-line [profile lab-app] block from ~/.aws/config in your editor
```

> If your default credentials come from Identity Center, set `AWS_PROFILE` to that profile name before running the lab, so `source_profile` points at it.

---

## Check yourself

<details><summary>Where does an app on EC2 get its AWS credentials if nothing is configured?</summary>From the instance metadata service, via the role in the instance profile. The SDK finds it automatically and refreshes it.</details>
<details><summary>What two policies does every role have?</summary>A trust policy (who may assume it) and permission policies (what it can do).</details>
<details><summary>Why does attaching a role to a Lambda function require iam:PassRole?</summary>So that nobody can give a service a role more powerful than their own permissions allow (privilege escalation).</details>
<details><summary>S3 must invoke your Lambda function. Whose policy needs to change?</summary>The Lambda function's resource-based policy must allow the s3.amazonaws.com principal (scoped with source ARN and account).</details>

---
**Previous:** [Module 01](01-how-access-works.md) · **Next:** [Module 03 — Policies in Detail](03-policies-in-detail.md)
