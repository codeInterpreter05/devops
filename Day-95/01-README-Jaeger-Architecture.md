# Day 95 — Jaeger & Tempo: Jaeger Architecture & Storage Backends

**Phase:** 3 – Observability | **Week:** W16 | **Domain:** Tracing

## Brief

OpenTelemetry (Day 94) tells you *how to generate and ship* traces. Jaeger is one of the two most common places those traces actually *land, get stored, and get queried*. Understanding Jaeger's internal architecture — not just "run the all-in-one Docker image" — matters because every production deployment eventually has to answer "why is trace ingestion falling behind," "why did old traces disappear," or "which storage backend should we pick," and those are architecture questions, not UI questions. This is also foundational for understanding Grafana Tempo (file 2), which solves the same problem with a very different, cheaper storage philosophy.

## Why a dedicated tracing backend exists

A trace is fundamentally a **write-heavy, high-cardinality, short-lived-value dataset**: potentially millions of spans per minute, each with unique IDs, rarely queried by exact match except when debugging (usually queried by service+operation+time-range+tags), and typically only useful for days-to-weeks before being safe to discard. This access pattern is a poor fit for a general-purpose database — Jaeger exists specifically to (a) accept extremely high write throughput and (b) support "find traces where service=X, operation=Y, duration>500ms, tag=Z, in the last hour" queries efficiently.

## Jaeger's components

Jaeger (originally built at Uber, donated to CNCF) is split into distinct services, each independently scalable:

- **jaeger-agent** *(legacy, being phased out in favor of the OTel Collector)* — a lightweight daemon that used to run on every host, receiving spans over UDP from local processes and batching them onward. Most new deployments skip this and have applications/OTel SDKs send directly to the Collector via OTLP instead (see Day 94).
- **jaeger-collector** — receives spans (via OTLP, or legacy Jaeger/Zipkin protocols), validates them, and writes them to the configured storage backend. This is the horizontally-scalable ingestion tier — add more collector replicas to absorb more write throughput. It also does some in-flight processing (e.g., adaptive sampling hints) and can apply span processors.
- **Storage backend** — a pluggable persistence layer (see below). Jaeger itself is stateless at the collector/query layer; all durability is delegated to this.
- **jaeger-query** — the read path: a service that translates UI/API queries ("all traces for `checkout-service` slower than 500ms in the last hour") into storage-backend-specific queries, and serves the Jaeger UI.
- **jaeger-ui** — the web frontend: search, trace timeline/waterfall view, service dependency graph (derived from span parent-child relationships across services), comparison view.
- **Spark/Flink dependency processor** *(optional, batch)* — periodically analyzes stored spans to build the service dependency graph shown in the UI, since that graph isn't cheap to compute live from raw spans at query time.

In the "all-in-one" Docker image used for local dev/labs, all of these run in a single process with an in-memory storage backend — convenient for a laptop, useless for production (data vanishes on restart, no durability, no horizontal scale).

## Storage backends — the real architectural decision

Jaeger doesn't mandate a specific database; you choose based on your scale/ops tradeoffs:

| Backend | Characteristics | When to choose it |
|---|---|---|
| **Cassandra** | Jaeger's original, most battle-tested backend at very large scale (this is what Uber runs). Write-optimized, horizontally scalable, but operationally heavy (you're now running a Cassandra cluster). | Very high trace volume, existing in-house Cassandra expertise. |
| **Elasticsearch/OpenSearch** | Also widely used; gives you flexible ad-hoc querying over span tags via ES's search capabilities, and many orgs already run ES for log storage, so it's one less system to operate. | You already run ELK/OpenSearch for logs and want to reuse that operational investment. |
| **In-memory** | No persistence, capped size, single process only. | Local dev/demo/lab only — this is what the all-in-one image uses by default. |
| **Badger (embedded)** | Local, embedded key-value store, no external dependency. | Small single-node/edge deployments. |
| **Grafana Tempo** *(not literally a "Jaeger storage backend," but the modern alternative)* | Object-storage-backed (S3/GCS/Azure Blob), no dedicated database cluster to run at all. | Covered fully in file 2 — usually the better default choice today for new deployments. |

**The operational tradeoff to internalize:** Cassandra/Elasticsearch give you a full secondary-index query engine (query by any tag), but you now own a stateful, sharded database cluster — capacity planning, compaction tuning, node failures, all of it. This operational cost is *exactly* the pain point Tempo was built to eliminate (file 2).

## Reading a trace in the Jaeger UI

When you open a trace, you see a **waterfall/flame graph**: each span rendered as a horizontal bar, its horizontal position/width representing start time and duration, nested under its parent. This is the single most useful view for latency debugging because it visually answers "which specific hop, out of N services this request touched, ate most of the total time" — a question that's nearly impossible to answer from logs or metrics alone. The **service dependency graph** view (built from the batch dependency processor) shows which services call which, useful for understanding blast radius before a deploy or during incident triage ("what else could be affected if service X is unhealthy").

## Points to Remember

- Jaeger splits into collector (write path, horizontally scalable ingestion), query (read path), UI (frontend), and a pluggable storage backend — durability lives entirely in the storage layer, not the collector/query services.
- jaeger-agent (UDP-based) is legacy; modern deployments send via OTLP straight to jaeger-collector (or through an OTel Collector first).
- Storage backend choice is the real architecture decision: Cassandra (battle-tested, heavy ops), Elasticsearch/OpenSearch (reuse existing log infra), in-memory (dev only, no persistence), or migrate to Tempo entirely (file 2).
- The all-in-one Docker image is single-process + in-memory — perfect for a lab, never for production (no durability, no horizontal scale).
- The waterfall view answers "which hop ate the time"; the service dependency graph answers "what talks to what" for blast-radius reasoning.

## Common Mistakes

- Running the Jaeger all-in-one image in a staging/production-adjacent environment and being surprised traces vanish on every pod restart — it's in-memory storage by default, meant for laptops only.
- Picking Elasticsearch as the storage backend without capacity-planning for the very different write pattern of spans vs. logs (much higher document rate, different retention needs) — leading to a cluster tuned for logs choking on trace volume.
- Treating jaeger-agent as still the recommended ingestion path in a new deployment — most new setups should send OTLP directly to jaeger-collector or via an OTel Collector, since jaeger-agent is legacy and UDP-based (unreliable, size-limited).
- Not setting a retention/TTL policy on the storage backend, letting trace data grow unbounded — traces are usually only valuable for days-to-weeks, and the storage bill (and query slowdown) compounds indefinitely without a purge policy.
- Assuming the service dependency graph updates in real time — it's typically computed by a periodic batch job (e.g. hourly Spark job), so it lags reality and shouldn't be trusted as a live topology view during an active incident.
