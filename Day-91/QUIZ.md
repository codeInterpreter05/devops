# Day 91 — Quiz: Thanos & Long-term Metrics

Try to answer without looking at your notes. Answers are at the bottom.

1. What is Prometheus's default retention period, and what two resource constraints (beyond "we set a config value") actually limit how long you can realistically retain data locally?
2. What is cardinality, and why does it primarily threaten Prometheus's memory, not just its disk?
3. Give a concrete example of a label that should never be added to a metric, and explain what happens if it is.
4. Which two Prometheus queries would you run to actively detect a cardinality explosion before it causes an outage?
5. What does the Thanos Sidecar do, specifically (two responsibilities)?
6. What's the difference between what the Sidecar serves and what the Store Gateway serves?
7. Why must only one Thanos Compactor run against a given object storage bucket at a time?
8. What problem does downsampling solve, and what are Thanos's default downsampling thresholds?
9. How does Thanos Query handle two HA Prometheus replicas scraping the same targets, and what label makes that possible?
10. Architecturally, what is the core difference between how Thanos and VictoriaMetrics achieve long-term storage?
11. Name two concrete reasons a team might choose VictoriaMetrics over Thanos, and one reason they might stick with Thanos.
12. **Interview question:** How do you store Prometheus metrics for 1 year across 10 clusters without spending a fortune?

---

## Answers

1. 15 days by default. Beyond the config value itself, you're bounded by (a) local disk space on whatever node/volume Prometheus runs on — there's no built-in horizontal scaling — and (b) query performance/compaction cost, since a single-node in-memory index and local TSDB weren't designed to efficiently scan years of chunks.
2. Cardinality is the number of distinct time series a metric produces (the product of distinct values across all its labels). Prometheus keeps metadata for every actively-scraped series in its in-memory head block and index, so memory usage scales primarily with the *number of active series*, not the volume of sample data — a modest disk footprint can still coincide with an OOM if series count is high.
3. Labels like `user_id`, `request_id`, `session_id`, or a raw unbounded URL path segment (e.g., `/orders/12345` instead of `/orders/{id}`). Adding one multiplies the series count by however many distinct values that label takes — for IDs, effectively unbounded and growing forever, which can OOM the Prometheus instance.
4. `prometheus_tsdb_head_series` (total active series right now) and `count by (__name__)({__name__=~".+"})` (series count broken down per metric name, to find which specific metric is exploding).
5. (1) Exposes the local Prometheus's data over Thanos's StoreAPI gRPC interface so Thanos Query can pull recent data from it. (2) Uploads completed, immutable TSDB blocks to object storage (S3/GCS/etc.) on a schedule.
6. The Sidecar serves *recent* data still living on the local Prometheus's disk. The Store Gateway serves *historical* data that has already been uploaded to and now lives only in object storage — it fetches blocks/chunks from the bucket on demand, it doesn't hold the data itself.
7. Concurrent compaction of the same blocks by multiple Compactor instances can corrupt the data — it's explicitly documented as unsafe. Redundancy for the Compactor should come from it being stateless/restartable, not from running multiple copies at once.
8. Downsampling precomputes lower-resolution rollups (5-minute resolution after 40 hours, 1-hour resolution after 10 days by default) so that wide time-range queries (e.g., a year-long dashboard panel) don't need to scan raw 15-second-resolution data, which would be far too slow. Thanos Query automatically picks the appropriate resolution for the requested range.
9. It deduplicates using a `replica` external label set differently on each HA replica (e.g., `replica: "0"` and `replica: "1"`) — Thanos Query recognizes series identical except for that label and merges them into one logical series (also filling gaps in one replica using the other), configured via `--query.replica-label=replica`.
10. Thanos is additive: existing Prometheus instances keep scraping/storing locally, and Thanos adds an object-storage tier plus a federated query layer on top via Sidecar/Store Gateway/Compactor/Query. VictoriaMetrics replaces the storage/query backend entirely — Prometheus (or another agent) `remote_write`s samples directly into VictoriaMetrics's own purpose-built, horizontally-scalable storage engine (vminsert/vmstorage/vmselect).
11. VictoriaMetrics reasons: generally lower CPU/RAM footprint for equivalent workloads, and fewer distinct component types to operate (simpler operational model) — plus it doesn't strictly require an object storage dependency. Thanos reason: a team already deeply invested in kube-prometheus-stack and with mature existing object storage can adopt it with minimal architectural change, keeping their existing Prometheus instances exactly as they are.
12. Strong answer: "I'd solve this with either Thanos or VictoriaMetrics, since plain Prometheus can't retain a year locally without huge per-cluster disk and has no cross-cluster query story. With Thanos, each cluster's Prometheus gets a Sidecar that uploads completed blocks to a shared, cheap object storage bucket (S3), a Compactor downsamples older data so year-scale queries stay fast, and a single Thanos Query fans PromQL out across all 10 clusters' Sidecars/Store Gateways and merges the results into one global view. With VictoriaMetrics, each cluster's Prometheus would instead `remote_write` into a shared VictoriaMetrics cluster, which handles long-term storage and cross-cluster querying itself with generally lower resource cost. Either way, the cost control comes from moving cold data off expensive local block storage onto object storage or a purpose-built TSDB, plus downsampling old data instead of keeping everything at full resolution forever — and I'd also make sure cardinality is under control first, since neither solution fixes an underlying unbounded-cardinality problem, it just changes where the pain shows up."
