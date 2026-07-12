# Day 43 — AWS Monitoring & Cost: CloudWatch Metrics, Alarms & Dashboards

**Phase:** 1 – Core DevOps | **Week:** W7 | **Domain:** AWS | **Flag:** –

## Brief

CloudWatch is the nervous system of an AWS account — every managed service (EC2, RDS, ELB, EKS, Lambda) pushes metrics into it automatically, and it's the backbone of both alerting and autoscaling. If you can't read a CloudWatch dashboard and reason about *why* an alarm fired, you can't operate production AWS infrastructure, full stop. This is also one of the fastest ways interviewers separate "used AWS in a tutorial" from "ran production AWS" — because the gotchas here (metric resolution, alarm evaluation windows, missing-data handling) only bite once you've been on call.

This day is split into three files:

1. **This file** — CloudWatch Metrics, Alarms, and Dashboards.
2. **[02-README-AWS-Config-Trusted-Advisor.md](02-README-AWS-Config-Trusted-Advisor.md)** — compliance and recommendation tooling.
3. **[03-README-Cost-Management.md](03-README-Cost-Management.md)** — Cost Explorer, Budgets, Savings Plans vs. RIs.

## CloudWatch Metrics

A metric is a **time-ordered set of data points** identified by a **namespace** (`AWS/EC2`, `AWS/RDS`, a custom namespace like `MyApp/Orders`), a **metric name** (`CPUUtilization`), and a set of **dimensions** (key-value pairs like `InstanceId=i-0123...`). The same metric name can exist many times over — once per unique combination of dimensions.

**Resolution and retention are linked, and this trips people up constantly:**

| Data point age | Retention | Resolution |
|---|---|---|
| < 3 hours | Kept | 1-second (high-resolution only) |
| < 15 days | Kept | 1-minute |
| < 63 days | Kept | 5-minute |
| < 455 days (15 months) | Kept | 1-hour |

This means if you `GetMetricData` for a 6-month-old window and ask for 1-minute granularity, CloudWatch **silently returns 1-hour aggregated points** instead — it doesn't error, it just gives you coarser data than you expected. Always check what period your query actually returned.

**Standard vs. custom metrics:**
- AWS services publish *standard* metrics for free at 1 or 5-minute resolution (EC2 detailed monitoring costs extra for 1-minute; basic is 5-minute).
- You publish **custom metrics** via `PutMetricData` (or, better, the **Embedded Metric Format (EMF)** — a JSON structure you write to stdout/CloudWatch Logs that the Lambda/ECS/EC2 agent automatically extracts into metrics, avoiding extra API calls and cost).
- **Metric Math** lets you combine existing metrics into derived ones on the fly (e.g., `errorRate = (errors / requests) * 100`) without publishing a new metric — computed at query/dashboard time.
- Statistics matter: `Average` hides outliers. For latency, always look at **p99/p95 percentiles**, not just average — a service can have a perfectly fine average latency while 1% of requests are timing out.

## CloudWatch Alarms

An alarm watches a metric over an **evaluation period** and transitions between three states: `OK`, `ALARM`, `INSUFFICIENT_DATA`.

```
Alarm fires when: N out of M evaluation periods breach the threshold
```
Example: "3 out of 3 periods of 5 minutes where CPUUtilization > 80%" means the condition must hold continuously for 15 minutes before the alarm goes to `ALARM` — this exists specifically to avoid flapping on transient spikes.

**Treat Missing Data** is a setting people forget exists, and it silently changes behavior:
- `missing` (default): doesn't affect alarm state — if a data point never arrives, that period just doesn't count either way.
- `notBreaching`: treats missing data as "good" — useful for metrics that only emit on errors (no data = no errors = fine).
- `breaching`: treats missing data as bad — useful for a health-check heartbeat metric where "the metric stopped arriving" itself means something is dead.
- `ignore`: keeps the alarm in its last state.

Getting this wrong is a classic silent-failure bug: an alarm on a heartbeat metric configured with the default `missing` setting will **never fire** if the process crashes and stops emitting entirely, because "no data" isn't treated as a breach.

**Composite alarms** combine multiple alarms with AND/OR/NOT logic (`ALARM(cpu-high) AND ALARM(latency-high)`) — used to reduce alert fatigue by only paging when multiple signals agree something is actually wrong, instead of one flaky metric spamming on-call.

**Alarm actions** — what happens when an alarm state changes: publish to an **SNS topic** (email/Slack/PagerDuty via subscription), trigger an **Auto Scaling policy** (scale out/in), or an **EC2 action** (reboot/stop/terminate/recover an instance).

**Anomaly Detection alarms** use a machine-learned band (based on historical patterns, including daily/weekly seasonality) instead of a static threshold — useful for metrics with natural cyclical variance (e.g., traffic that's always higher on weekdays) where a static threshold would either miss real problems or false-positive constantly.

## Dashboards

Dashboards are JSON-defined collections of widgets (line/stacked graphs, number, gauge, text, alarm status, logs table). Key practical points:
- Widgets can pull from **multiple regions and even multiple accounts** in one dashboard (cross-account observability, set up via CloudWatch's "sharing" feature or AWS Organizations) — essential when you run a multi-account landing zone and need one pane of glass.
- Dashboards are **not** the alerting mechanism — they're for humans looking, not automated response. Don't confuse "I have a dashboard for X" with "I'll get paged if X breaks."
- **Container Insights** (for EKS/ECS) auto-generates a rich set of dashboards and metrics (node/pod/cluster CPU, memory, network) once you deploy the CloudWatch agent + Fluent Bit as a DaemonSet — this is usually the fastest path to real EKS observability without deploying Prometheus/Grafana from scratch.

## CloudWatch Logs & Logs Insights

Logs are organized into **log groups** (usually one per application/service, e.g., `/aws/eks/my-cluster/application`) containing **log streams** (usually one per instance/pod/task). Two settings people forget:
- **Retention defaults to "Never expire"** — logs accumulate forever and quietly become a real cost line item. Set an explicit retention (`aws logs put-retention-policy --log-group-name X --retention-in-days 30`) on every log group you create.
- **Subscription filters** stream log events in near-real-time to a Lambda, Kinesis stream, or OpenSearch — this is how you build cross-account log aggregation or feed logs into a SIEM.
- **Metric filters** turn a pattern match in your logs into a CloudWatch metric (e.g., count of lines containing `ERROR`) — cheap way to alarm on log content without a full log-analysis pipeline.

**Logs Insights** is a purpose-built query language for ad-hoc log analysis without exporting anywhere:

```
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 20

fields @message
| filter ispresent(statusCode)
| stats count(*) as requests by statusCode
| sort requests desc

stats avg(latency), pct(latency, 99) as p99 by bin(5m)
```

Queries only scan the log groups you explicitly select and the time range you specify — cost is per-GB-scanned, so narrowing the time window and log group selection matters for both speed and cost.

## Points to Remember

- Metric resolution degrades with age: 1-min data becomes 5-min after 15 days, then 1-hour after 63 days — a query on old data may silently return coarser granularity than expected.
- Alarms need N-out-of-M breaching periods to fire — this is intentional debouncing, not a bug.
- `Treat Missing Data` defaults to `missing` (no effect) — for heartbeat/liveness metrics, you almost always want `breaching` instead, or the alarm will never fire when the process dies.
- Dashboards are for humans; alarms + SNS/PagerDuty are for automated paging — don't rely on "someone will notice the dashboard."
- Log group retention defaults to forever — always set it explicitly, or logs silently become a cost problem.
- For latency-type metrics, always check p95/p99, not just average — averages hide the tail that actually causes user-facing pain.

## Common Mistakes

- Setting a CPU alarm threshold without considering evaluation periods, so a single 30-second spike (that Average smooths over anyway) never even reaches alarm — or conversely, setting 1-out-of-1 periods and getting paged for every noise blip.
- Forgetting to set `Treat Missing Data` to `breaching` on a "my service sends a heartbeat metric" alarm, so a fully crashed service silently reports `INSUFFICIENT_DATA` forever instead of paging anyone.
- Leaving CloudWatch Logs retention on "Never expire" across dozens of log groups, then getting a surprise bill months later — this is one of the most common "invisible" AWS cost leaks.
- Using `Average` as the only statistic for latency dashboards, missing that p99 latency has quietly doubled while the average looks flat.
- Assuming a CloudWatch dashboard *is* alerting — nobody gets paged just because a graph turned red on a screen nobody is watching at 3 a.m.
