# Module 02 — Logs & Alarms

← [All tutorials](../README.md) · **Observability tutorial**, module 2 of 3

Metrics tell you **that** something is wrong: errors went up. **Logs** tell you **what** happened: which request, which error message. **Alarms** make sure you find out without staring at a dashboard. This module covers both, with a query and an alarm definition explained piece by piece.

---

## 1. CloudWatch Logs

| Concept | What it is | Example |
|---|---|---|
| **Log event** | One line or record, with a timestamp | `{"level":"error","msg":"db timeout","orderId":"o-88213"}` |
| **Log stream** | The events from **one** source, in order | One container, one EC2 instance, one Lambda instance |
| **Log group** | A collection of streams that share settings (**retention**, encryption, access) | `/ecs/shop-api`, `/aws/lambda/make-thumbnail` |

How logs get there:
- **AWS services** write them for you: Lambda, ECS (the `awslogs` driver, see [ECS 02](../ecs/02-task-definition-field-by-field.md)), VPC Flow Logs, API Gateway, RDS (optional).
- On **EC2**, the **CloudWatch agent** ships log files such as `/var/log/app.log`.
- Your code can also call `PutLogEvents`, but writing to stdout and letting the platform ship it is simpler.

**Set retention on every log group.** The default is *never expire*, so you pay to store every log forever. Common choices: 30 days for application logs, 1 year (or longer, in S3) for audit logs. **Log structured JSON** rather than free text, so you can query fields. Use the **Infrequent Access** log class for logs you rarely search; it's cheaper to ingest.

### Logs Insights: querying logs, line by line

Logs Insights is a query language for log groups. Here's a query that finds the slowest checkout requests in the last hour, from JSON logs:

```sql
fields @timestamp, route, latency_ms, status
| filter route = "/checkout" and (status >= 500 or latency_ms > 1000)
| stats count(*) as slow_or_failed, pct(latency_ms, 95) as p95 by bin(5m)
| sort slow_or_failed desc
| limit 20
```

| Line | Meaning |
|---|---|
| `fields @timestamp, route, latency_ms, status` | Which fields to work with. Fields starting with `@` are built in (`@timestamp`, `@message`, `@logStream`). JSON keys are discovered automatically |
| `filter …` | Keep only matching events. Supports `=`, `!=`, `<`, `>`, `and`, `or`, `like /regex/`, `in [...]` |
| `stats count(*) … , pct(latency_ms, 95) … by bin(5m)` | Aggregate: count events and compute the 95th percentile **per 5-minute bucket**. Other functions: `avg`, `sum`, `min`, `max`, `count_distinct` |
| `sort slow_or_failed desc` | Order the results |
| `limit 20` | Return at most 20 rows |

For unstructured text, `parse` extracts fields: `parse @message "user=* action=*" as user, action`. You pay per GB of data scanned, so narrow the time range and the log groups.

### Getting value out of logs

- **Metric filter:** turn a log pattern into a metric, e.g. count lines containing `ERROR`. You can then graph and **alarm** on it (lab below).
- **Subscription filter:** stream log events in real time to Lambda, Kinesis Data Firehose (to S3 or a SIEM), or OpenSearch.
- **Live Tail:** watch logs arrive in real time in the console (or with `aws logs tail --follow`).
- **Data protection policies:** automatically mask sensitive data such as emails or card numbers in logs.

---

## 2. CloudWatch Alarms

An **alarm** watches **one metric** (or a metric math expression) and changes state when the metric crosses a threshold. Here's a complete alarm definition, the input of `aws cloudwatch put-metric-alarm`:

```json
{
  "AlarmName": "shop-alb-5xx-high",
  "AlarmDescription": "More than 10 server errors per minute on the shop load balancer",
  "Namespace": "AWS/ApplicationELB",
  "MetricName": "HTTPCode_Target_5XX_Count",
  "Dimensions": [{ "Name": "LoadBalancer", "Value": "app/shop-alb/50dc6c495c0c9188" }],
  "Statistic": "Sum",
  "Period": 60,
  "EvaluationPeriods": 5,
  "DatapointsToAlarm": 3,
  "Threshold": 10,
  "ComparisonOperator": "GreaterThanThreshold",
  "TreatMissingData": "notBreaching",
  "AlarmActions": ["arn:aws:sns:eu-central-1:111122223333:on-call"],
  "OKActions":    ["arn:aws:sns:eu-central-1:111122223333:on-call"]
}
```

| Field | Meaning |
|---|---|
| `AlarmName`, `AlarmDescription` | Names are unique per Region. Write the description for the person woken up at 3 a.m. |
| `Namespace`, `MetricName`, `Dimensions` | **Which** metric. Exactly the same three parts as in Module 01 |
| `Statistic`, `Period` | How each data point is computed: the **sum** of 5xx responses per **60-second** bucket |
| `EvaluationPeriods: 5`, `DatapointsToAlarm: 3` | Look at the **last 5 data points**, and go to `ALARM` if **at least 3** of them breach. This **"3 out of 5"** pattern ignores a single spike but catches a real problem |
| `Threshold`, `ComparisonOperator` | "Breaching" means `> 10`. Other operators: `GreaterThanOrEqualToThreshold`, `LessThanThreshold`, `LessThanOrEqualToThreshold` (and band operators for anomaly detection) |
| `TreatMissingData` | What a missing data point counts as: `notBreaching` (good when no traffic means no errors), `breaching`, `ignore`, or `missing`. **Decide deliberately:** for a "heartbeat" metric, missing data *is* the problem |
| `AlarmActions` | What happens when the alarm goes to **ALARM**: usually an **SNS topic** (which sends email, chat messages, or pages via PagerDuty), an Auto Scaling policy, or an EC2 action (reboot/recover) |
| `OKActions` | What happens when it returns to **OK**, e.g. "resolved" notifications |

**Alarm states:**
- `OK`: within the threshold.
- `ALARM`: breaching, per the rules above.
- `INSUFFICIENT_DATA`: not enough data yet, e.g. right after creation, or missing data treated as `missing`.

**Beyond simple thresholds:**
- **Metric math alarms:** alarm on an expression such as `100 * errors / requests > 5` (an error *rate* scales with traffic; a count doesn't).
- **Anomaly detection:** CloudWatch learns the normal daily and weekly pattern of a metric, and alarms when it leaves that band. Good for traffic-shaped metrics.
- **Composite alarms:** combine alarms with AND/OR, for example "page only if 5xx is high **and** latency is high". This reduces noise a lot.

---

## 3. Try it: logs → metric filter → alarm

```bash
LG=/lab/observability
aws logs create-log-group --log-group-name $LG
aws logs put-retention-policy --log-group-name $LG --retention-in-days 1
aws logs create-log-stream --log-group-name $LG --log-stream-name app-1

# Metric filter: every log line containing ERROR adds 1 to Lab/AppErrors
aws logs put-metric-filter --log-group-name $LG --filter-name errors --filter-pattern ERROR \
  --metric-transformations metricName=AppErrors,metricNamespace=Lab,metricValue=1,defaultValue=0

# The alarm, built from a JSON definition like the one above: >= 3 errors in one minute
cat > /tmp/alarm.json <<'EOF'
{"AlarmName":"lab-app-errors","AlarmDescription":"3+ ERROR lines per minute in /lab/observability",
 "Namespace":"Lab","MetricName":"AppErrors","Statistic":"Sum","Period":60,
 "EvaluationPeriods":1,"DatapointsToAlarm":1,"Threshold":3,"ComparisonOperator":"GreaterThanOrEqualToThreshold",
 "TreatMissingData":"notBreaching"}
EOF
aws cloudwatch put-metric-alarm --cli-input-json file:///tmp/alarm.json

# Write some log events (timestamps in milliseconds)
NOW=$(($(date +%s)*1000))
aws logs put-log-events --log-group-name $LG --log-stream-name app-1 --log-events \
  "[{\"timestamp\":$NOW,\"message\":\"INFO started\"},
    {\"timestamp\":$((NOW+1)),\"message\":\"ERROR db timeout\"},
    {\"timestamp\":$((NOW+2)),\"message\":\"ERROR db timeout\"},
    {\"timestamp\":$((NOW+3)),\"message\":\"ERROR db timeout\"}]" >/dev/null

# Query them with Logs Insights
QID=$(aws logs start-query --log-group-name $LG --start-time $(($(date +%s)-900)) --end-time $(($(date +%s)+60)) \
  --query-string 'filter @message like /ERROR/ | stats count(*) as errors' --query queryId --output text)
sleep 5; aws logs get-query-results --query-id $QID --query 'results'

# Within 1–3 minutes, the alarm moves from INSUFFICIENT_DATA/OK to ALARM
for i in $(seq 1 8); do aws cloudwatch describe-alarms --alarm-names lab-app-errors \
  --query 'MetricAlarms[0].[StateValue,StateReason]' --output text; sleep 30; done

# Clean up
aws cloudwatch delete-alarms --alarm-names lab-app-errors
aws logs delete-log-group --log-group-name $LG
rm -f /tmp/alarm.json
```

---

## Check yourself

<details><summary>What's the cheapest single fix for a large CloudWatch Logs bill?</summary>Set retention on every log group, and cut noisy debug logging.</details>
<details><summary>How do you get alerted on ERROR lines in your logs?</summary>A metric filter on the log group turns them into a metric, and an alarm on that metric notifies an SNS topic.</details>
<details><summary>What do EvaluationPeriods = 5 and DatapointsToAlarm = 3 mean together?</summary>The alarm fires if 3 of the last 5 data points breach the threshold.</details>
<details><summary>Should a heartbeat metric (sent every minute by a job) treat missing data as notBreaching?</summary>No. For a heartbeat, missing data means the job stopped, so use `breaching`.</details>

---
**Previous:** [Module 01](01-metrics.md) · **Next:** [Module 03 — Traces & Audit](03-traces-and-audit.md)
