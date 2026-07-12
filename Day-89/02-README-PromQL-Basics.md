# Day 89 — Prometheus Fundamentals: PromQL Basics & Recording Rules

**Phase:** 3 – Observability | **Week:** W15 | **Domain:** Metrics | **Flag:** ⚡ Interview-critical

## Brief

PromQL is the query language you'll write more of than any other observability syntax in this phase — every dashboard panel, alert rule, and SLO burn-rate calculation you build downstream (Grafana on Day 90, alerting throughout) is a PromQL expression underneath. The functions in this file — `rate()`, `increase()`, `histogram_quantile()` — are the three you'll type constantly, and getting their semantics wrong (especially around windows and counter resets) is one of the fastest ways to ship a dashboard that quietly lies to whoever's on call.

## `rate()` — per-second average rate of increase

```promql
rate(http_requests_total{status="500"}[5m])
```

`rate()` takes a **counter** and a **range vector** (the `[5m]` part) and returns the **per-second average rate of increase** over that window, extrapolated to cover the full range. It automatically detects and compensates for **counter resets** (e.g., the process restarted and the counter dropped back to 0) — it does this by assuming any decrease between two adjacent samples means a reset happened, and adjusts the calculation instead of returning a nonsensical negative rate.

Why 5 minutes and not 1 minute? **Rule of thumb: your range should be at least 4x your scrape interval.** With a 15s scrape interval, a `[1m]` window only contains ~4 data points — too few for `rate()` to average out noise, and if you happen to miss even one scrape the result can swing wildly or briefly report no data. `[5m]` with a 15s interval gives you ~20 samples, smoothing transient spikes while still reacting to real changes within a few minutes.

## `increase()` — total increase over a window

```promql
increase(http_requests_total{status="500"}[1h])
```

`increase()` is `rate()` multiplied by the number of seconds in the range — it answers "how many *total* errors happened in the last hour" rather than "what's the errors-per-second rate." Internally it's computed the same way (extrapolated from the rate), so it inherits the same counter-reset handling and the same need for a reasonably-sized window. Use `rate()` for graphs and alert thresholds expressed as a rate (e.g., "more than 5 errors/sec"); use `increase()` when you want a human-readable total ("143 errors in the last hour").

## `histogram_quantile()` — approximate percentiles from bucketed data

```promql
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
)
```

This is the canonical p99/p95 latency query pattern, and it's worth understanding every piece:

1. `rate(..._bucket[5m])` — get the per-second rate of observations falling into each bucket, per time series (per pod/instance).
2. `sum(...) by (le, service)` — aggregate across all instances of a service, **grouping by `le`** (you must keep `le` in the `by` clause, or `histogram_quantile()` has nothing to interpolate across) and by whatever other label you care about (`service`, `route`, etc.).
3. `histogram_quantile(0.99, ...)` — given the cumulative bucket counts, linearly interpolate the value below which 99% of observations fall.

**Why this only works because histograms are cumulative:** `histogram_quantile()` needs to know, for each bucket boundary, how many total observations fell at or below it — that's exactly what the `le` (less-than-or-equal) cumulative buckets give it. It walks the buckets to find which one crosses the 99th-percentile count, then linearly interpolates within that bucket's range. This is an **approximation** — if your buckets are coarse (e.g., jumping from 0.5s to 5s with nothing in between) and your real p99 sits at 2s, the interpolated answer can be significantly off. Always design bucket boundaries around your actual SLO thresholds.

## A few more PromQL essentials

```promql
# Aggregation operators
sum(rate(http_requests_total[5m])) by (status)
avg(node_load1) by (instance)
max(container_memory_usage_bytes) by (pod)
count(up == 0)                          # how many targets are down

# Binary operators + vector matching
sum(rate(http_requests_total{status=~"5.."}[5m]))
  /
sum(rate(http_requests_total[5m]))       # error ratio — classic RED-method query

# offset — compare to the past
rate(http_requests_total[5m]) offset 1d  # same query, this time yesterday

# label_replace — reshape labels for joins
label_replace(up, "job_name", "$1", "job", "(.*)")
```

The error-ratio pattern above (`sum(rate(errors[5m])) / sum(rate(total[5m]))`) is the single most common alerting query in production — it's the numerator/denominator pattern behind almost every "error rate %" panel and alert you'll ever write.

## Recording rules

Recording rules **precompute and store expensive or frequently-used PromQL expressions** as new time series, evaluated on a schedule (usually matching your scrape interval) rather than re-computed every time a dashboard loads or an alert fires.

```yaml
groups:
  - name: api-slo-rules
    interval: 30s
    rules:
      - record: job:http_request_duration_seconds:p99
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, job)
          )
      - record: job:http_requests:error_ratio5m
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (job)
          /
          sum(rate(http_requests_total[5m])) by (job)
```

Why this matters operationally:
- **Performance**: a `histogram_quantile()` over a `sum(rate(...))` across hundreds of pods is expensive to compute on every dashboard refresh, especially at a 15s scrape interval with a busy Grafana instance polling every panel every 30s. Precomputing it once, on the same schedule Prometheus already scrapes, amortizes that cost.
- **Consistency**: alert rules and dashboards reference the *same* recorded metric, so they never disagree because one used a `[5m]` window and the other a `[10m]` window.
- **Naming convention**: the community convention is `level:metric:operations`, e.g. `job:http_requests:error_ratio5m` — reads as "aggregated by job, metric is http_requests, operation is a 5-minute error ratio." Following this convention makes recorded metrics self-documenting when someone else queries them later.

## Points to Remember

- `rate()` = per-second average increase, handles counter resets automatically; needs a window ≥ 4x the scrape interval to be statistically meaningful.
- `increase()` = `rate()` × window duration; use it for human-readable totals, `rate()` for graphs/alert thresholds.
- `histogram_quantile()` requires the `le` label to survive any `sum by (...)` aggregation — drop it and the function has nothing to interpolate over.
- The error-ratio pattern `sum(rate(errors[5m])) / sum(rate(total[5m]))` is the backbone of most production alerting.
- Recording rules precompute expensive/frequent queries on a schedule — use them for anything referenced by both a dashboard and an alert.

## Common Mistakes

- Using a `[1m]` (or shorter) range with `rate()` on a 15–30s scrape interval — too few samples, noisy/flaky results, occasional "no data" gaps.
- Forgetting `by (le)` when aggregating histogram buckets before `histogram_quantile()` — the query either errors or silently returns garbage.
- Applying `rate()` to a gauge (nonsensical) or querying a raw counter without `rate()`/`increase()` (a monotonically increasing line since process start tells you nothing actionable).
- Writing the same complex `histogram_quantile(sum(rate(...)))` expression separately in five dashboard panels and three alert rules, then having them silently drift apart when someone edits one but not the others — this is exactly what recording rules exist to prevent.
- Not aligning recording-rule `interval` with the underlying scrape interval, producing recorded series that are staler or choppier than expected.
