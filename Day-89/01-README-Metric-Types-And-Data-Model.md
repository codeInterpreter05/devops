# Day 89 — Prometheus Fundamentals: Data Model & Metric Types

**Phase:** 3 – Observability | **Week:** W15 | **Domain:** Metrics | **Flag:** ⚡ Interview-critical

## Brief

Prometheus is the de-facto standard for metrics in cloud-native systems — if a job posting mentions Kubernetes, there's almost certainly Prometheus (or a Prometheus-compatible system like VictoriaMetrics or Mimir) behind it. Before you can write a single useful query, you need to understand *what a metric actually is* to Prometheus: how it's identified, what shape its data takes over time, and — critically — which of the four metric types you should reach for. Picking the wrong metric type (e.g., a gauge for something that should be a counter) is one of the most common real-world mistakes that quietly breaks dashboards and alerts months later.

This day is split into four focused files:

1. **This file** — the Prometheus data model and the four metric types.
2. **[02-README-PromQL-Basics.md](02-README-PromQL-Basics.md)** — `rate()`, `increase()`, `histogram_quantile()`, and recording rules.
3. **[03-README-Scrape-Configs-And-Service-Discovery.md](03-README-Scrape-Configs-And-Service-Discovery.md)** — how Prometheus finds and pulls metrics from targets.
4. **[04-README-Alertmanager.md](04-README-Alertmanager.md)** — routes, receivers, and inhibition rules.

## The Prometheus data model

Every Prometheus data point is a **time series**: a stream of `(timestamp, float64 value)` samples, uniquely identified by a **metric name** plus a set of **key-value labels**.

```
http_requests_total{method="GET", path="/api/orders", status="200", instance="10.0.1.5:9100"}
```

That entire string — metric name + label set — is one time series. Change *any* label value, and you get a **different** time series stored separately. This is the single most important mental model in Prometheus: labels aren't metadata bolted onto one series, they're what defines series identity. `http_requests_total{method="GET"}` and `http_requests_total{method="POST"}` are two completely independent counters that happen to share a name for query-grouping purposes.

Prometheus is a **pull-based** system: it scrapes an HTTP endpoint (conventionally `/metrics`) on each target at a configured interval (commonly 15s or 30s) and stores whatever samples it finds at that instant, tagged with the scrape timestamp. This is different from push-based systems (StatsD, some APM agents) where the app pushes metrics to a collector — pull-based scraping gives Prometheus built-in target health (`up{job=...}` becomes 0 the instant a scrape fails) without any extra plumbing.

## The four metric types

Prometheus's client libraries expose four fundamental types. The wire format (what actually gets scraped) only really distinguishes counters/gauges from histograms/summaries, but conceptually all four exist and picking the right one matters enormously.

### Counter

A value that **only ever goes up** (or resets to 0 on process restart). Used for anything you're *counting* — total requests served, total errors, total bytes sent.

```
# HELP http_requests_total Total number of HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",status="200"} 27183
```

You almost never query a counter's raw value directly (27183 total requests since the process started is not actionable). Instead you always wrap it in `rate()` or `increase()` to get a *per-second rate* or *increase over a window* — covered in the next file. If you find yourself graphing a raw counter and it looks like a sawtooth that resets to zero periodically, that's a **counter reset** (process restarted) — `rate()` is specifically designed to detect and handle this correctly, which is why you should never compute rate-of-change manually with subtraction.

### Gauge

A value that can go **up or down** — current memory usage, number of active connections, queue depth, temperature. You query gauges directly; there's no "rate" concept that makes sense for most gauges (though you can still apply functions like `delta()` or `deriv()` for trend analysis).

```
# TYPE node_memory_MemAvailable_bytes gauge
node_memory_MemAvailable_bytes 2147483648
```

### Histogram

Samples observations (typically request durations or response sizes) into **configurable buckets**, and exposes three things per histogram: a `_bucket{le="..."}` series per bucket (cumulative count of observations ≤ that bucket's upper bound), a `_sum` (running total of all observed values), and a `_count` (total number of observations).

```
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{le="0.1"} 8000
http_request_duration_seconds_bucket{le="0.5"} 9500
http_request_duration_seconds_bucket{le="1"} 9800
http_request_duration_seconds_bucket{le="+Inf"} 10000
http_request_duration_seconds_sum 2450.3
http_request_duration_seconds_count 10000
```

Buckets are **cumulative** (`le` = "less than or equal to") — the `le="0.5"` bucket includes everything counted in `le="0.1"` plus new observations between 0.1s and 0.5s. This cumulative design is exactly what lets `histogram_quantile()` interpolate an approximate percentile after the fact, on the server side, from data that was aggregated cheaply on the client side. The tradeoff: precision is bounded by your bucket boundaries — if all your traffic falls between two adjacent buckets, your quantile estimate is only as good as linear interpolation between them. Choosing good bucket boundaries (matching your actual latency distribution and SLOs) is a real design decision, not an afterthought.

### Summary

Also tracks distributions, but computes **quantiles client-side** (in the application process) using a sliding time window, and exposes them directly as `{quantile="0.5"}`, `{quantile="0.99"}`, etc., plus `_sum` and `_count`.

```
# TYPE http_request_duration_seconds summary
http_request_duration_seconds{quantile="0.5"} 0.042
http_request_duration_seconds{quantile="0.99"} 0.31
http_request_duration_seconds_sum 2450.3
http_request_duration_seconds_count 10000
```

**Why histograms are usually preferred over summaries:** summary quantiles **cannot be aggregated across instances**. If you have 20 pods each exposing a `p99` summary quantile, you cannot average or combine those 20 numbers into a meaningful cluster-wide p99 — quantiles simply don't compose that way mathematically. Histogram buckets *can* be summed across instances (`sum(rate(..._bucket[5m])) by (le)`) and then run through `histogram_quantile()` once, giving a mathematically valid cluster-wide percentile. This is why almost every modern instrumentation guide (and the Prometheus docs themselves) recommend histograms over summaries for anything you intend to aggregate — which, in a multi-replica Kubernetes world, is essentially everything.

## Points to Remember

- A time series = metric name + full label set. Changing one label value creates an entirely new, independently-stored series.
- Prometheus is pull-based: it scrapes `/metrics` on a schedule; a failed scrape shows up automatically as `up == 0` for that target.
- Counters only increase (reset to 0 on restart) — never query them raw, always wrap in `rate()`/`increase()`.
- Gauges can go up or down and are queried directly (current value, or `delta()`/`deriv()` for trend).
- Histograms bucket observations cumulatively (`le`) and support server-side `histogram_quantile()`; buckets **can** be summed across instances.
- Summaries compute quantiles client-side and **cannot** be meaningfully aggregated across instances — prefer histograms when you'll have multiple replicas.

## Common Mistakes

- Using a gauge to track something that should be a counter (e.g., "total errors" as a gauge that gets reset to 0 by the app itself) — this breaks `rate()`-based alerting because the app's own reset logic fights with Prometheus's counter-reset detection.
- Trying to average or sum `quantile` values from a summary across multiple pod replicas — mathematically invalid; you need histograms summed at the bucket level instead.
- Choosing histogram bucket boundaries that don't match the actual latency distribution (e.g., buckets at 0.1/0.5/1/5s when your service's real p99 sits at 800ms) — this makes `histogram_quantile()` output wildly inaccurate even though the query "works."
- Adding a high-cardinality label (like `user_id` or a raw request path with IDs embedded, e.g. `/orders/12345`) to a counter or histogram — every unique label combination is a brand-new time series, and this is the single most common cause of a Prometheus instance running out of memory (see Day 91's cardinality discussion).
- Forgetting that `up{job="..."}` exists and instead building custom "is this target alive" metrics from scratch.
