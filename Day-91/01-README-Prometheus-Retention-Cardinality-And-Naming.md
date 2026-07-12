# Day 91 — Thanos & Long-term Metrics: Retention Limits, Cardinality & Naming

**Phase:** 3 – Observability | **Week:** W15 | **Domain:** Metrics | **Flag:** —

## Brief

Before reaching for Thanos or VictoriaMetrics, you need to understand *why* plain Prometheus can't just be configured to "keep data forever" and *why* it sometimes falls over long before you even hit a retention limit. Both problems — storage growth from retention, and memory blowup from cardinality — come from the same root cause: Prometheus is fundamentally a **single-node, local-disk, in-memory-index** time series database. Interviewers probe cardinality specifically because it's the #1 real-world cause of "Prometheus OOMKilled itself" incidents, and it's caused by application-level instrumentation decisions, not by Prometheus infrastructure — meaning an SRE has to be able to explain it to *application developers*, not just fix it in a Helm chart.

This day is split into three focused files:

1. **This file** — retention limits, cardinality, and metric naming best practices.
2. **[02-README-Thanos-Architecture.md](02-README-Thanos-Architecture.md)** — Sidecar, Store Gateway, Compactor, Query, and the global view across clusters.
3. **[03-README-VictoriaMetrics-Alternative.md](03-README-VictoriaMetrics-Alternative.md)** — VictoriaMetrics as an alternative long-term-storage approach.

## Why Prometheus has retention limits at all

By default Prometheus retains **15 days** of data (`--storage.tsdb.retention.time`), and it's a **single binary with local disk storage** — no built-in replication, no built-in horizontal scaling, no object storage support natively. Retention is bounded by:
- **Disk space** on the node/volume Prometheus runs on — there's no "just add more nodes" scaling story, only "attach a bigger disk," which has real limits (and if that node/pod dies, you lose everything on it unless you've solved durability yourself).
- **Query performance** — even if disk were infinite, query latency degrades as the on-disk index and number of chunks Prometheus has to scan for a wide time range grows; a single-node in-memory index wasn't designed for years of retention.
- **Compaction cost** — Prometheus's local TSDB periodically compacts smaller blocks into larger ones; extremely long local retention means larger and larger compaction workloads competing with the ingestion/query workload on the same box.

You *can* raise `--storage.tsdb.retention.time` to 1 year, but you're then betting your organization's historical metrics on one disk, one pod, with no cross-cluster query story — which is exactly the gap Thanos/VictoriaMetrics (next two files) exist to close.

## Cardinality — why it kills Prometheus

**Cardinality** = the number of distinct time series a metric produces, which is the product of the number of distinct values across *all* of its labels. A metric with 3 labels each taking 100 possible values has up to 100 × 100 × 100 = 1,000,000 possible series — and Prometheus has to hold metadata for **every actively-scraped series** in memory (its in-memory head block + index), regardless of how rarely any individual series is queried.

Why this actually kills a Prometheus instance in practice, not just theoretically:
- Every unique label combination is a **new time series** that must be tracked, indexed, and stored — even if its value never changes.
- Adding a label like `user_id`, `request_id`, `session_id`, or an **unbounded** path segment (`/orders/12345` instead of `/orders/:id`) to a counter or histogram doesn't add "a bit more data" — it can multiply the series count by however many distinct values that label takes, which for user/request IDs is effectively unbounded and grows forever as new users/requests appear.
- Prometheus's memory usage scales primarily with the **number of active series**, not the volume of raw sample data — this is why a Prometheus instance can OOM even though its disk usage looks modest: the in-memory head block and series index are what blow up.
- This is exactly why histograms are especially dangerous cardinality multipliers: each histogram metric already produces N+2 series (N buckets + `_sum` + `_count`) *per unique label combination* — a histogram with a high-cardinality label multiplies that N+2 by however many label combinations exist.

**Real mitigation strategies:**
- Never put unbounded, user-controlled, or per-entity-ID values in a label. Use them in **logs** (Day 92) instead, where per-line cardinality doesn't compound the same way.
- Normalize dynamic path segments before they become labels (`/orders/{id}` as a label value, not `/orders/12345`) — usually done in application middleware/instrumentation, not in Prometheus itself.
- Use `metric_relabel_configs` (Day 89, file 3) to drop or rewrite an already-existing high-cardinality label at scrape time if you can't fix the source immediately.
- Monitor cardinality itself: `prometheus_tsdb_head_series` (total active series) and `count by (__name__)({__name__=~".+"})` (series count per metric name) are the queries you run *on* Prometheus to catch a cardinality problem before it takes the instance down.

## Metric naming best practices

Prometheus's own conventions (documented in its "Metric and label naming" guide) exist because consistent naming makes metrics discoverable and predictable across an entire organization:

- **Base unit + `_unit` suffix**: `http_request_duration_seconds` (always base SI units — seconds not milliseconds, bytes not kilobytes) so cross-metric math and `histogram_quantile()` outputs are always in a predictable, comparable unit.
- **`_total` suffix for counters**: `http_requests_total` — signals at a glance "this only goes up, wrap it in `rate()`."
- **Namespacing prefix**: `<application>_<subsystem>_<name>_<unit>`, e.g. `checkout_payment_processing_duration_seconds` — avoids collisions once you have hundreds of services all emitting metrics into one Prometheus.
- **Labels for dimensions, not for identity**: `status_code` as a label on `http_requests_total` is correct (a bounded, small set of values); a metric literally named `http_requests_total_500` (baking the status into the metric name) is an anti-pattern — it breaks the ability to aggregate across status codes with a single query.

## Points to Remember

- Prometheus's default retention (15d) and local-disk, single-node design are why "just retain it forever" isn't viable without an external long-term-storage layer.
- Cardinality = distinct label-value combinations; Prometheus memory scales with **active series count**, not raw sample volume — this is why cardinality, not disk size, is the more common cause of outright Prometheus failures.
- Never label a metric with user IDs, request IDs, session IDs, or raw unbounded path segments — normalize them or move that detail into logs instead.
- `prometheus_tsdb_head_series` and `count by (__name__)(...)` are how you actively monitor for a cardinality problem before it OOMs the instance.
- Naming conventions (`_unit` suffix, `_total` for counters, `<app>_<subsystem>_<name>_<unit>` namespacing) exist to keep metrics predictable and safely aggregatable across a large, multi-team Prometheus deployment.

## Common Mistakes

- Adding a `user_id`, `email`, or `request_id` label "just in case it's useful for debugging" — this is the single most common cause of a Prometheus instance running out of memory, and it's almost always an application-instrumentation decision, not something the ops team caused.
- Baking a dynamic identifier into the metric *name* instead of a label (e.g., `orders_created_customer_42_total`) — breaks aggregation entirely and multiplies the number of distinct metric names instead of series, which is arguably worse for discoverability.
- Believing "cardinality" only matters for extremely large organizations — a single histogram with even a moderately-sized ID label (a few thousand distinct values) times several buckets can already produce hundreds of thousands of series, well within reach of a mid-sized team's default Prometheus instance.
- Raising `--storage.tsdb.retention.time` to cover a year "because we need history" without considering query performance or durability — a single Prometheus pod with a year of local data is both slow to query over wide ranges and a single point of data loss if the volume is lost.
- Not naming metrics with a unit suffix, then having a `duration` metric where nobody can tell (without reading the instrumentation code) whether it's in seconds or milliseconds — leading to `histogram_quantile()` outputs that are off by 1000x and dashboards that look "wrong" for reasons nobody can explain quickly.
