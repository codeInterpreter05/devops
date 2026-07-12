# Day 99 — AWS-native Observability I: Container Insights & Logs Insights

**Phase:** 3 – Observability | **Week:** W17 | **Domain:** Observability | **Flag:** ⚡ Interview-critical

## Brief

If your workloads run on EKS and you're not paying for a self-hosted metrics/logging stack, CloudWatch is almost always the first tool in the loop — it's already wired into every AWS account, IAM-authenticated, and zero-ops. Container Insights and Logs Insights are the two features that turn raw CloudWatch from "a place logs go to die" into an actual observability surface for Kubernetes. Interviewers probe this because it separates people who've only used Prometheus/Grafana on a laptop from people who've operated AWS-native stacks in a real account with real billing.

This day is split into three focused files:

1. **This file** — CloudWatch Container Insights for EKS, and the CloudWatch Logs Insights query language.
2. **[02-README-Xray-And-ADOT-Tracing.md](02-README-Xray-And-ADOT-Tracing.md)** — distributed tracing with X-Ray and AWS Distro for OpenTelemetry (ADOT).
3. **[03-README-Synthetics-And-Managed-Prometheus-Grafana.md](03-README-Synthetics-And-Managed-Prometheus-Grafana.md)** — synthetic monitoring with CloudWatch Synthetics, and the managed vs. self-managed Prometheus/Grafana decision.

## CloudWatch Container Insights for EKS

Container Insights is CloudWatch's purpose-built dashboard and metric collection layer for containerized workloads (ECS, EKS, self-managed Kubernetes on EC2). For EKS specifically, it works one of two ways:

- **Legacy path**: you deploy the **CloudWatch agent** and **Fluent Bit** as DaemonSets yourself (via a Quick Start manifest or Helm chart). The CloudWatch agent scrapes cAdvisor/kubelet stats on each node; Fluent Bit tails container logs and ships them to CloudWatch Logs.
- **Modern path**: the **`amazon-cloudwatch-observability` EKS add-on** — a single `aws eks create-addon` call installs and manages both components for you, including "Container Insights with enhanced observability," which adds control-plane metrics (API server, etcd, scheduler) and richer per-pod resource attribution without you hand-rolling DaemonSets.

```bash
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name amazon-cloudwatch-observability \
  --addon-version v2.4.0-eksbuild.1 \
  --resolve-conflicts OVERWRITE
```

**What actually gets collected:** CPU, memory, disk, and network utilization at cluster → namespace → node → pod → container granularity, published as CloudWatch **custom metrics** under namespaces like `ContainerInsights` and `ECS/ContainerInsights`. The pre-built "Container Insights" dashboard in the console renders these automatically — no dashboard-building required on day one.

**Why this matters for cost:** CloudWatch bills custom metrics per metric-per-month (roughly the same PutMetricData pricing model whether it's a StatsD counter you emit or a Container Insights dimension). Because Container Insights creates a metric *per container* per resource type, a cluster with thousands of short-lived pods can generate a surprisingly large metrics bill — this is the single most common "why is our CloudWatch bill so high" surprise for teams that turned on Container Insights without thinking about cardinality. Compare this to Prometheus, where storage cost is a function of your own infrastructure, not a per-series AWS bill — a genuinely different cost model, not just a different price point (covered in more depth in file 3).

## CloudWatch Logs Insights query language

Logs Insights is **not SQL** — it's a purpose-built pipe-based query language for ad-hoc querying of one or more CloudWatch Logs log groups. The mental model: each query scans the raw log events in your selected time range (it does not maintain a persistent index the way Elasticsearch does), extracts fields, and pipes them through a chain of operations.

Core syntax:

```
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 20
```

- `fields` — select/project fields (built-ins are `@timestamp`, `@message`, `@logStream`, `@log`).
- `filter` — like a `WHERE` clause; supports `=`, `!=`, `>`, `like`, `in`, boolean `and`/`or`.
- `parse` — regex/glob-style extraction from unstructured text into new fields, e.g. `parse @message "user=* status=*" as user, status`.
- `stats` — aggregation: `count()`, `avg()`, `sum()`, `pct(field, 99)` for percentiles, usually combined with `by` for grouping and `bin()` for time-bucketing.

A realistic example — p99 latency per API route over 5-minute buckets, from JSON-structured app logs:

```
fields @timestamp, @message
| filter ispresent(latency_ms)
| stats pct(latency_ms, 99) as p99, count() as requests by route, bin(5m)
```

An error-rate-over-time query, useful as a saved query wired into a dashboard widget:

```
fields @timestamp
| filter level = "ERROR"
| stats count() as errors by bin(1m)
```

**Why time-range matters so much:** because Logs Insights scans data rather than reading a pre-built index, both **cost and query latency scale with the amount of log data scanned** in your chosen time window, not with how selective your filter is. A `filter` on a 30-day window still scans 30 days of raw bytes before discarding non-matches. Narrow the time range first, then filter — this is the single biggest lever for both speed and cost. Saved queries (visible under "Queries" in the Logs Insights console) let a team standardize the good queries instead of everyone re-deriving `parse` patterns from scratch.

## Points to Remember

- Container Insights on EKS can be deployed manually (CloudWatch agent + Fluent Bit DaemonSets) or via the `amazon-cloudwatch-observability` add-on, which also unlocks control-plane metrics.
- Container Insights metrics are billed as CloudWatch custom metrics — high pod cardinality/churn directly drives cost; this is the most common surprise EKS teams hit after enabling it.
- Logs Insights is a scan-based query language, not an indexed search engine — narrow the time range before adding filters, because cost and latency are driven by bytes scanned, not by filter selectivity.
- `stats ... by bin(5m)` is the idiomatic pattern for time-series aggregation (rate, percentile, count) inside Logs Insights.
- Saved queries are the practical way to make good Logs Insights queries reusable across a team instead of tribal knowledge.

## Common Mistakes

- Enabling Container Insights on a high-churn, high-pod-count EKS cluster without checking the CloudWatch custom-metrics cost impact first, then being surprised by the bill at month-end.
- Writing a Logs Insights query with a broad time range (e.g., "last 7 days") "just to be safe," which multiplies scan cost and query time for no benefit if the actual incident window is 20 minutes.
- Treating Logs Insights like SQL and expecting joins across log groups or persistent indexes — it supports querying multiple log groups in one query, but there's no cross-log-group join semantics like a relational database.
- Forgetting that `parse` patterns are order- and format-sensitive — a single log format change (e.g., a new log line prefix from an updated library) silently breaks every saved query built on `parse`.
- Assuming the legacy CloudWatch agent + Fluent Bit DaemonSet setup and the newer `amazon-cloudwatch-observability` add-on are interchangeable with zero side effects — running both at once causes duplicate metrics and doubled ingestion cost.
