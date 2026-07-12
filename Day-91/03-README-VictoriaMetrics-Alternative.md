# Day 91 — Thanos & Long-term Metrics: VictoriaMetrics as an Alternative

**Phase:** 3 – Observability | **Week:** W15 | **Domain:** Metrics | **Flag:** —

## Brief

Thanos isn't the only answer to "long-term, multi-cluster Prometheus storage" — VictoriaMetrics (and similarly, Grafana Mimir/Cortex) takes a fundamentally different architectural approach: instead of a set of sidecar/gateway/compactor components sitting alongside existing Prometheus instances and object storage, it's a **purpose-built, horizontally-scalable time series database** that Prometheus (or any Prometheus-remote-write-compatible agent) writes into directly. Knowing both options — and specifically *why* someone would pick one over the other — is what turns "I've heard of Thanos" into an actual system-design answer.

## Architectural difference from Thanos

Thanos is fundamentally additive: it wraps existing Prometheus instances (Sidecar), adds an object-storage tier (Compactor, Store Gateway), and adds a federated query layer (Query) on top. The underlying Prometheus instances keep doing exactly what they always did — local scraping, local alerting evaluation, local short-term storage.

VictoriaMetrics instead **replaces** the storage/query backend: Prometheus (or Grafana Agent, vmagent, Telegraf, etc.) is configured to `remote_write` its samples directly into VictoriaMetrics, which stores and indexes them in its own custom, highly compressed storage engine. In its clustered form, VictoriaMetrics splits into:
- **vminsert** — receives incoming remote-write samples and distributes them across storage nodes.
- **vmstorage** — the actual storage nodes holding time series data, horizontally scalable by adding more nodes.
- **vmselect** — handles queries (PromQL-compatible, plus its own extended **MetricsQL**), fanning out across `vmstorage` nodes and merging results.

This is architecturally closer to how a distributed database like Cassandra or a dedicated TSDB (InfluxDB, TimescaleDB) is built — purpose-designed for ingest and storage at scale — rather than "Prometheus plus a set of add-on components."

## Why teams choose VictoriaMetrics over Thanos

- **Resource efficiency**: VictoriaMetrics is widely benchmarked (including by its own maintainers, though also independently) as using significantly less RAM and CPU than both Prometheus itself and Thanos for equivalent ingest/query workloads, largely due to its storage engine design and compression. For cost-sensitive or very high-cardinality environments, this matters directly on the cloud bill.
- **Operational simplicity**: fewer distinct component types to run and reason about than Thanos's Sidecar + Store Gateway + Compactor + Query (plus object storage and its own IAM/networking concerns) — VictoriaMetrics's clustered mode has three component types with a more conventional "distributed database" mental model.
- **Long-term storage without object storage dependency**: VictoriaMetrics can run entirely on local/block storage with its own replication, whereas Thanos's long-term story is fundamentally built around an S3-compatible object store — if your environment doesn't have good object storage (or you want to avoid the egress/API cost model of cloud object storage), VictoriaMetrics's model may fit better.
- **Drop-in `remote_write` target**: since Prometheus natively supports `remote_write`, adopting VictoriaMetrics doesn't require Sidecars or new sidecar containers — just a `remote_write` block pointed at VictoriaMetrics, keeping existing Prometheus scrape configs and alerting untouched.

```yaml
# prometheus.yml
remote_write:
  - url: http://vminsert:8480/insert/0/prometheus/
```

## Why teams still choose Thanos

- **Already deeply invested in the Prometheus Operator / kube-prometheus-stack ecosystem** — Thanos Sidecar integrates as a natural sidecar to existing `Prometheus` CRDs with minimal architectural change; you keep exactly the Prometheus instances you already have.
- **Strong existing object storage investment** (mature S3/GCS setup, cost-optimized lifecycle policies already in place) — Thanos leans into that rather than requiring a new dedicated storage tier to operate.
- **CNCF-graduated, very widely adopted** — for organizations prioritizing "boring, well-trodden, widely-documented" choices, Thanos has a longer track record specifically in the Kubernetes/Prometheus ecosystem context.

## The honest "how would you answer this in an interview" framing

For "how do you store Prometheus metrics for a year across 10 clusters without spending a fortune," a strong answer names **both** approaches and the actual tradeoff axis, rather than picking one dogmatically:
- Both solve the same two problems: durable long-retention storage cheaper than local SSD, and a federated/global query view across clusters.
- Thanos: additive, keeps existing Prometheus instances as-is, relies on object storage + downsampling for cost efficiency, more moving component types.
- VictoriaMetrics: replaces the storage/query backend via `remote_write`, generally lower resource footprint, fewer component types, doesn't require an object storage dependency (though it can be configured with backup-to-object-storage too, that's a separate feature from its core storage engine).
- The actual deciding factors in a real job would be: existing infrastructure investment (object storage maturity, whether you're already deep in kube-prometheus-stack), team size/operational appetite for running more component types, and measured resource cost for your specific cardinality/retention/query-pattern profile — worth explicitly saying you'd benchmark both against real workload characteristics rather than assume.

## Points to Remember

- Thanos is additive (Sidecar/Store Gateway/Compactor/Query wrapped around existing Prometheus + object storage); VictoriaMetrics is a replacement storage/query backend that Prometheus writes into via `remote_write`.
- VictoriaMetrics's clustered components: `vminsert` (ingest/distribute), `vmstorage` (horizontally-scalable storage), `vmselect` (query/merge) — a more conventional distributed-database shape than Thanos's set of add-on components.
- VictoriaMetrics is commonly chosen for lower resource footprint and operational simplicity; Thanos is commonly chosen when already deeply invested in kube-prometheus-stack and mature object storage.
- MetricsQL (VictoriaMetrics's query language) is a PromQL superset — existing PromQL queries generally work unmodified, with additional functions available.
- A strong interview answer names the tradeoff axis (existing infra investment, operational appetite, measured resource cost) rather than declaring one tool universally better.

## Common Mistakes

- Presenting Thanos or VictoriaMetrics as a strict drop-in replacement for Prometheus itself — both still rely on Prometheus (or a remote-write-compatible agent) to do the actual scraping; they solve storage/query federation, not scraping.
- Assuming `remote_write` to VictoriaMetrics (or any remote system) doesn't need backpressure/retry consideration — under load or network partition, Prometheus's remote-write queue can back up and drop samples if not tuned (`queue_config` settings), a common surprise for teams who "just add remote_write" without reading the tuning docs.
- Picking a long-term-storage architecture based purely on hype/what a blog post recommended, without benchmarking against your own actual cardinality and query patterns — the "right" choice is genuinely workload-dependent, and both projects publish very different-looking benchmark numbers depending on what's measured.
- Forgetting that adopting either solution doesn't remove the need to fix upstream cardinality problems (previous file) — both systems will still struggle (just with different failure characteristics) if fed genuinely unbounded-cardinality metrics.
