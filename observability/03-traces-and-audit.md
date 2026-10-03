# Module 03 — Traces & Audit: X-Ray, CloudTrail, AWS Config

← [All tutorials](../README.md) · **Observability tutorial**, module 3 of 3

Two questions remain from Module 01:
- **Where is the time going?** A checkout takes 3 seconds and passes through the load balancer, the API, the orders service, the database, and a payment provider. Metrics say "slow", but not **where**. **Traces** do.
- **Who changed what?** A security group rule disappeared last night. **CloudTrail** records every AWS API call, and **AWS Config** records how each resource's configuration changed over time.

---

## 1. Distributed tracing

A **trace** follows **one request** through every service it touches:
- Each step is a **span**, with a start time, a duration, and attributes: "orders-service → `SELECT` on `orders` → 840 ms".
- A **trace ID** travels with the request (in an HTTP header), so spans from different services join up into one trace.

```text
Trace 1-67a8…  GET /checkout                                     total 3,050 ms
├─ ALB                                                              12 ms
└─ shop-api   POST /checkout                                     3,020 ms
   ├─ orders-service  CreateOrder                                  910 ms
   │  └─ PostgreSQL  INSERT INTO orders                            840 ms   ← slow query
   └─ payments  POST https://api.payment-provider.com/charge       2,050 ms  ← external API is the main cost
```

How you get traces on AWS:
- **Instrument** your code with **OpenTelemetry** (the open standard), usually through auto-instrumentation libraries, so you don't write span code by hand. AWS ships the **AWS Distro for OpenTelemetry (ADOT)**, including a collector that runs as a sidecar container or an agent.
- Traces are stored and viewed in **AWS X-Ray**. **CloudWatch Application Signals** builds a **service map** from them and lets you define **SLOs** (service level objectives, e.g. "99.9% of checkouts succeed in under 1 s").
- **Sample** traces (e.g. 5% of requests, plus every error) to keep costs predictable.

---

## 2. CloudTrail: who did what

**CloudTrail** records every API call made in your account, from the console, the CLI, SDKs, and AWS services acting for you. The last **90 days** of **management events** (creating, changing, deleting resources) are viewable for free in **Event history**.

### A CloudTrail event, field by field

Someone removed a security group rule. This is the event CloudTrail recorded (trimmed):

```json
{
  "eventTime": "2025-10-02T22:41:07Z",
  "eventSource": "ec2.amazonaws.com",
  "eventName": "RevokeSecurityGroupIngress",
  "awsRegion": "eu-central-1",
  "sourceIPAddress": "203.0.113.45",
  "userAgent": "aws-cli/2.17.20 Python/3.11 Darwin/24.0.0",
  "userIdentity": {
    "type": "AssumedRole",
    "arn": "arn:aws:sts::111122223333:assumed-role/AWSReservedSSO_Developer_3f2a1b/bob@mycompany.com",
    "accountId": "111122223333",
    "sessionContext": {
      "sessionIssuer": { "type": "Role", "arn": "arn:aws:iam::111122223333:role/aws-reserved/sso.amazonaws.com/AWSReservedSSO_Developer_3f2a1b" },
      "attributes": { "creationDate": "2025-10-02T21:58:12Z", "mfaAuthenticated": "true" }
    }
  },
  "requestParameters": {
    "groupId": "sg-0a1b2c3d4e5f60718",
    "ipPermissions": { "items": [{ "ipProtocol": "tcp", "fromPort": 5432, "toPort": 5432,
                                   "groups": { "items": [{ "groupId": "sg-0app0a1b2c3d4e5f6" }] } }] }
  },
  "responseElements": { "_return": true },
  "errorCode": null,
  "eventID": "6f1c2b0e-…",
  "readOnly": false
}
```

| Field | What it tells you |
|---|---|
| `eventTime` | **When** (always UTC) |
| `eventSource`, `eventName` | **Which service and API**: EC2's `RevokeSecurityGroupIngress`. The same names you use in IAM policies as `ec2:RevokeSecurityGroupIngress` ([IAM 03](../iam/03-policies-in-detail.md)) |
| `awsRegion` | Where the call was made |
| `sourceIPAddress`, `userAgent` | **From where, with what**: an IP address and the AWS CLI on a Mac. Console actions show the browser, and AWS services show their own name (`ecs.amazonaws.com`) |
| `userIdentity.type` / `arn` | **Who**: the Identity Center role `Developer`, **session `bob@mycompany.com`**. The session name identifies the actual person behind a role, which is why SSO sessions are so useful for auditing ([IAM 01](../iam/01-how-access-works.md)) |
| `sessionContext.attributes.mfaAuthenticated` | Whether the session was signed in with MFA |
| `requestParameters` | **Exactly what** was asked: remove TCP 5432 from the database's security group `sg-0a1b…` for source `sg-0app…`. The app servers just lost database access |
| `responseElements`, `errorCode` | Whether it **worked**. A denied call shows `errorCode: "AccessDenied"` here, which is useful for finding permission problems and suspicious activity |
| `readOnly` | `true` for Describe/List/Get calls, `false` for changes |

### Beyond event history

- **Create a trail** (for a whole AWS Organization) to keep events **longer** in an S3 bucket, encrypted and protected from changes, and query them with **Athena** or **CloudTrail Lake**.
- **Data events** (S3 object reads and writes, Lambda invocations, DynamoDB item access) are **off by default**, because they're high volume and billed. Enable them where you need object-level auditing.
- Send CloudTrail events to **EventBridge** to **react** in real time, e.g. notify a channel whenever someone changes a security group or disables MFA.

---

## 3. AWS Config: how resources changed

CloudTrail answers *who called which API*. **AWS Config** answers **what a resource looked like** at any time, and **whether it follows your rules**:
- It records a **configuration item** for each resource every time it changes. You can view a security group's rules as they were last Tuesday, and see which CloudTrail event changed them.
- **Config rules** continuously check compliance, e.g. "S3 buckets must block public access", "EBS volumes must be encrypted", "security groups must not allow 0.0.0.0/0 on port 22". Non-compliant resources are flagged, and optionally fixed automatically.
- Config charges per recorded change and per rule evaluation. Organizations usually enable it centrally.

---

## 4. Try it: find out who did what in your account

```bash
# The last 10 changes (non-read-only calls) made in this Region
aws cloudtrail lookup-events --lookup-attributes AttributeKey=ReadOnly,AttributeValue=false --max-results 10 \
  --query 'Events[].[EventTime,EventName,Username,EventSource]' --output table

# Everything a specific API did, e.g. who created or deleted security group rules recently
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=AuthorizeSecurityGroupIngress \
  --max-results 5 --query 'Events[].[EventTime,Username]' --output table

# Read one full event, and match its fields to the table above
aws cloudtrail lookup-events --max-results 1 --query 'Events[0].CloudTrailEvent' --output text \
  | python3 -m json.tool | head -40

# Recent failed calls (denied permissions show up here)
aws cloudtrail lookup-events --max-results 50 --query 'Events[].CloudTrailEvent' --output json \
  | python3 -c 'import json,sys; [print(e["eventTime"], e["eventName"], e["errorCode"]) for e in map(json.loads, json.load(sys.stdin)) if e.get("errorCode")]' \
  | head -5
```

If you did the other tutorials' labs recently, you'll see your own `CreateVpc`, `RunInstances`, or `CreateRole` calls, with your identity in `Username`.

---

## Check yourself

<details><summary>Metrics show checkout is slow. Which tool shows which service or query is responsible?</summary>Distributed tracing: X-Ray with OpenTelemetry instrumentation, and Application Signals for the service map.</details>
<details><summary>In a CloudTrail event, which fields tell you who made the call and how they signed in?</summary>`userIdentity` (type, ARN, and the session name of an assumed role) and `sessionContext.attributes.mfaAuthenticated`.</details>
<details><summary>Are S3 object downloads in CloudTrail event history by default?</summary>No. They're data events, which you must enable on a trail (and pay for).</details>
<details><summary>CloudTrail vs AWS Config?</summary>CloudTrail: who called which API, when, and from where. Config: what each resource's configuration was over time, and whether it complies with rules.</details>

---
**Previous:** [Module 02](02-logs-and-alarms.md) · **Back to:** [All tutorials](../README.md)
