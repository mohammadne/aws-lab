# Module 03 — Traces & Audit: X-Ray, CloudTrail, Config

← [All tutorials](../README.md) · **Observability** (short tutorial), module 3 of 3

---

## 1. Traces

- Instrument with **OpenTelemetry** (the AWS Distro for OpenTelemetry, **ADOT**) or the X-Ray SDK. Traces show each request's path across ALB → service → DB/queue, and where time is spent.
- **CloudWatch Application Signals** builds service maps and **SLOs** (availability, latency) from traces and metrics automatically.
- Sample traces (e.g. 5%) to control cost. Keep 100% of the errors.

## 2. Audit: CloudTrail & Config

- **CloudTrail** records **management events** (control-plane API calls: who, what, when, from where) for **90 days** in Event history at no cost. Create an **organization trail** to S3 for long-term retention and Athena queries. **Data events** (S3 object reads, Lambda invokes) are opt-in and cost extra.
- **AWS Config** records resource configuration over time and evaluates **rules** (e.g. "S3 buckets must block public access").
- **EventBridge** reacts to events in real time (e.g. ECS task stopped, GuardDuty finding → Lambda or Slack).

---

## Check yourself

<details><summary>Why can't you see EC2 memory usage in CloudWatch?</summary>The hypervisor can't see inside the guest OS. Install the CloudWatch agent (or use Container Insights for containers).</details>
<details><summary>How do you alert on "ERROR" lines in logs?</summary>A metric filter on the log group creates a metric, and an alarm on that metric notifies SNS.</details>
<details><summary>Someone deleted a security group rule yesterday. Where do you look?</summary>CloudTrail (RevokeSecurityGroupIngress event: who, when, from where). AWS Config shows the before and after.</details>
<details><summary>The cheapest single fix for a big CloudWatch Logs bill?</summary>Set retention on every log group, and reduce noisy debug logging.</details>

---
**Previous:** [Module 02](02-logs-and-alarms.md) · **Back to:** [All tutorials](../README.md)
