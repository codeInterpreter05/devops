# Day 91 — Thanos & Long-term Metrics: Thanos Architecture

**Phase:** 3 – Observability | **Week:** W15 | **Domain:** Metrics | **Flag:** —

## Brief

Thanos exists to answer two problems plain Prometheus can't solve alone: "how do I query metrics across 10 separate Prometheus instances/clusters as if they were one system" and "how do I keep years of metrics without paying for years of local SSD on every Prometheus node." Its architecture is a genuinely common interview topic in SRE/platform roles because "how would you store Prometheus metrics for a year across 10 clusters affordably" is a real, frequently-asked system-design question — and the honest answer requires knowing what each Thanos component actually does, not just "we added Thanos."

## The core problem, restated

A vanilla Prometheus setup is one Prometheus per cluster (sometimes per team), each with its own local retention window and no shared query surface. Two consequences:
1. **No global view** — "what's the error rate across all 10 production clusters" requires manually querying 10 different Prometheus UIs and adding numbers by hand.
2. **Retention is capped by local disk** — long-term trending/capacity-planning data (a year of history) either isn't kept at all, or requires an expensive amount of local SSD per Prometheus instance, per cluster, forever.

Thanos solves both by (a) putting a shared query layer in front of many Prometheus instances, and (b) offloading old data out of local Prometheus storage into cheap object storage (S3, GCS, etc.), which is virtually unlimited and far cheaper per GB than block storage.

## The components

### Sidecar

Runs **alongside each Prometheus instance** (literally a sidecar container in the same pod, in Kubernetes). It does two jobs:
1. Exposes Prometheus's data over Thanos's internal **StoreAPI** gRPC interface, so Thanos Query (below) can pull *recent* data straight out of that Prometheus's local storage without Prometheus itself needing to know Thanos exists.
2. **Uploads completed TSDB blocks to object storage** on a schedule (Prometheus writes immutable 2-hour blocks to disk; once a block is no longer being written to, the sidecar ships a copy to S3/GCS/Azure Blob).

The Sidecar is what turns "one Prometheus per cluster, each with local-only data" into "one Prometheus per cluster, still doing local scraping, but with its data also durably backed up and reachable through a shared API."

### Store Gateway

Serves historical data **that lives in object storage**, not on any Prometheus's local disk, back out through the same StoreAPI interface. When Thanos Query needs data older than what a live Sidecar/Prometheus still has locally, it asks the Store Gateway, which fetches the relevant blocks (or, more precisely, just the index/chunks it needs) from S3/GCS.

This is the component that actually delivers "1 year of retention without keeping 1 year of data on every Prometheus's local disk" — the data physically lives in object storage; Store Gateway is just the query-serving front for it.

### Compactor

Runs against the object storage bucket (not against any live Prometheus) and does three things:
1. **Compaction** — merges many small 2-hour blocks into larger blocks (same concept as Prometheus's own local compaction, just operating on the object-storage copies) to make long-range queries more efficient.
2. **Downsampling** — after a configurable age (Thanos defaults: downsample to 5-minute resolution after 40 hours, 1-hour resolution after 10 days), it precomputes lower-resolution aggregates and stores them alongside the raw data. Query then automatically selects the *appropriate resolution* for the requested time range — a dashboard panel showing a year of data doesn't need (and would be far too slow to render from) raw 15-second-resolution samples; it uses the 1-hour downsampled rollup instead, transparently.
3. **Retention enforcement** on the object-storage side — deleting blocks older than your configured long-term retention (which can be "forever," bounded only by storage budget).

**Only one Compactor should run per bucket at a time** — it's explicitly documented as not safe to run concurrently against the same storage path, because concurrent compaction of the same blocks can corrupt data.

### Query (Querier)

The component you (and Grafana) actually talk to. It implements the Prometheus HTTP query API, so from Grafana's perspective it's just "a Prometheus data source" — but under the hood, Query fans a single PromQL query out to every Sidecar and Store Gateway registered with it (typically discovered via DNS SD or a static list), merges the results (**deduplicating** overlapping data — see below), and returns one unified answer.

This is the piece that delivers the **global query view across clusters**: point Query at the Sidecars of 10 different clusters' Prometheus instances, and a single PromQL query executed against Thanos Query effectively runs against all 10 simultaneously, merged into one result set.

### Deduplication and HA pairs

A common Prometheus HA pattern runs **two identical Prometheus replicas** scraping the same targets (so a single Prometheus crash doesn't lose data or break alerting) — but this means Thanos Query would otherwise see two near-identical, slightly time-skewed copies of every series. Thanos Query supports **deduplication** via a `replica` label (each HA replica is scraped with an external label like `replica: "0"` / `replica: "1"`) — Query recognizes series that are identical except for that label and merges them into one logical series, picking up gaps in one replica from the other rather than showing duplicate/doubled data.

## Points to Remember

- **Sidecar** = lives next to a live Prometheus; serves recent local data via StoreAPI and uploads completed blocks to object storage.
- **Store Gateway** = serves historical data that lives in object storage (not on any Prometheus disk) back out through StoreAPI.
- **Compactor** = runs against the object storage bucket only; merges blocks, downsamples for query efficiency, and enforces long-term retention. Exactly one instance per bucket — concurrent compaction risks corruption.
- **Query** = the unified query entrypoint; implements the Prometheus API, fans queries out to Sidecars + Store Gateways, merges/deduplicates results — this is what gives you "one query across N clusters."
- Downsampling (5m after 40h, 1h after 10d by default) is what makes year-long dashboard queries fast — Query automatically selects the right resolution for the requested range.
- Deduplication via a `replica` external label is how Thanos handles HA Prometheus pairs without doubling data.

## Common Mistakes

- Running more than one Compactor against the same object storage bucket "for redundancy" — this is explicitly unsafe and can corrupt blocks; redundancy for the Compactor comes from it being stateless and restartable, not from running multiple copies concurrently.
- Forgetting to set a distinguishing `replica` external label on HA Prometheus pairs before enabling Thanos deduplication — without it, Query can't tell the two replicas apart from two genuinely different data sources and either double-counts or fails to merge correctly.
- Assuming Store Gateway "has" the data — it queries object storage on demand; if bucket access/permissions/network to S3/GCS are misconfigured, Store Gateway will fail queries for historical ranges while recent (Sidecar-served) data still works fine, which is a confusing partial-failure mode to debug without understanding the split.
- Not configuring downsampling and then wondering why a 1-year dashboard panel times out or takes minutes to render — it's trying to scan raw 15s-resolution data across a year instead of using precomputed rollups.
- Treating Thanos as "a drop-in Prometheus replacement" rather than "a layer that sits alongside existing Prometheus instances" — Prometheus itself keeps scraping and evaluating alerts locally exactly as before; Thanos adds global query and long-term storage on top, it doesn't replace the scraping/alerting engine.
