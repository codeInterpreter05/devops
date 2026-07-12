# Day 95 — Jaeger & Tempo: Grafana Tempo & Correlating the Three Pillars

**Phase:** 3 – Observability | **Week:** W16 | **Domain:** Tracing

## Brief

Grafana Tempo is the modern answer to Jaeger's storage problem: instead of running a Cassandra or Elasticsearch cluster just to store traces, Tempo stores trace data directly in cheap **object storage** (S3, GCS, Azure Blob, or local disk) and gives up full secondary indexing in exchange for radically simpler operations and lower cost. Combined with Grafana as the single-pane UI, Tempo is what makes "click a spike on a dashboard, land on the exact failing trace, then jump to its logs" a smooth reality instead of a manual, three-tool exercise — which is the actual point of "the three pillars of observability," and today's hands-on activity.

## Why Tempo exists: the storage-cost problem

Jaeger's Cassandra/Elasticsearch backends need a full secondary index so you can query traces by arbitrary tag (`http.status_code=500`, `user.id=42`, etc.) — that indexing is exactly what makes those databases expensive to run at scale (they must maintain index structures across a distributed, sharded, replicated cluster). Tempo's insight, shared by Grafana Labs' other "cheap object storage" projects (Loki for logs, Mimir for metrics), is: **you almost always find a trace by its `trace_id`** (extracted from a log line, a metric exemplar, or a user bug report), not by an arbitrary tag scan. So Tempo only needs a lightweight index (or none at all, depending on backend) mapping `trace_id -> object storage location`, and dumps the actual span data as compressed blocks straight into S3/GCS. This trades "rich ad-hoc tag search" for "10-100x cheaper storage and far simpler ops" — no database cluster to run, just object storage, which most orgs already operate reliably for other reasons.

## Tempo architecture

- **Distributor** — receives spans (OTLP, Jaeger, Zipkin protocols all supported), and consistently hashes them by `trace_id` to route all spans of one trace to the same **ingester**.
- **Ingester** — buffers incoming spans in memory, batches them into immutable **blocks**, and flushes completed blocks to object storage on a timer/size threshold.
- **Object storage** (S3/GCS/Azure Blob/local disk/MinIO) — the actual durable store. This is the entire "database" — no separate stateful cluster beyond the object store you already run.
- **Querier** — on a lookup request (by `trace_id`), finds and reads the relevant blocks from object storage (with help from an optional **compactor**-maintained index/bloom filters to avoid scanning everything).
- **Compactor** — merges small blocks into larger ones over time and enforces retention (deleting blocks past the configured TTL) — this is what keeps object storage from accumulating millions of tiny files indefinitely.

This mirrors Loki's architecture almost exactly (by design — same company, same philosophy) — if you already know Loki, Tempo's shape will feel familiar.

## The critical limitation: find-by-ID, not find-by-content

Tempo (in its default/simple mode) is optimized for **"give me trace X"**, not **"find me all traces where attribute Y = Z."** That second kind of query (full tag-based search) either isn't supported well natively or requires the newer **TraceQL** query layer with its own indexing (Tempo added TraceQL, a query language modeled after PromQL/LogQL, specifically to close this gap — but it's still fundamentally a scan-oriented, not fully-indexed, search, so query cost/latency scales with how much of the time range you're scanning). This is precisely why the exemplar/correlation-based workflow below is the primary way you're expected to *reach* a trace_id in the first place — not by searching Tempo's UI cold.

```
# Example TraceQL query — find traces with an error span on a specific service
{ resource.service.name = "checkout-service" && status = error }

# Filter by duration
{ duration > 500ms }

# Combine attribute + duration
{ span.http.status_code = 500 && duration > 1s }
```

## The three pillars, correlated: how you actually reach a trace_id

The real value of the traces+logs+metrics story isn't having three separate UIs — it's **jumping between them via shared identifiers**, typically wired up in Grafana like this:

1. **Metrics → Trace (exemplars).** Prometheus/Mimir histograms can attach **exemplars** — a sampled example data point with a `trace_id` attached — directly onto a metric time series. When you see a latency spike on a `histogram_quantile` panel in Grafana, you can click a specific exemplar dot and jump straight to that exact trace in Tempo, no manual searching.
2. **Trace → Logs.** Grafana's Tempo data source supports a "trace to logs" configuration: given a trace's `trace_id` (and optionally span time range), Grafana auto-generates a Loki query (e.g., `{service="checkout-service"} |= "<trace_id>"`) and shows you the exact log lines emitted during that span — assuming your app logs are structured to include `trace_id` (which OTel's logging integration does automatically when a log call happens inside an active span, see Day 94).
3. **Logs → Trace.** Conversely, if you're grepping Loki and find an error log line that includes a `trace_id` field, Grafana renders it as a clickable link straight into the corresponding Tempo trace.
4. **Trace → Metrics (span metrics / RED).** Tempo can generate **span metrics** (via the `metrics-generator` component) — aggregating span data into RED metrics (Rate, Errors, Duration) per service/operation, pushed to Prometheus/Mimir — so you get dashboard-level trend data derived automatically from the very traces you're already collecting, without instrumenting separate metrics by hand (this is covered in depth in file 3).

This closes the loop completely: a dashboard alert fires (metric) → click exemplar → land on the exact trace → click "logs for this span" → read the precise error → done. No manual timestamp-matching across three different tools' UIs.

## Points to Remember

- Tempo trades rich secondary indexing for object-storage economics — it's optimized for "fetch trace by ID," with TraceQL added on top for broader (but still scan-cost-bound) queries.
- Architecture mirrors Loki: distributor (hash by trace_id) → ingester (buffer + flush blocks) → object storage (the actual DB) → querier + compactor.
- The primary way you reach a `trace_id` in practice is correlation, not cold search: metric exemplars, or a `trace_id` embedded in a log line.
- OTel's logging integration auto-injects `trace_id`/`span_id` into log records emitted inside an active span — this is the mechanism that makes trace-to-logs linking possible at all; if your app doesn't structure logs this way, the link is dead on arrival.
- Tempo's `metrics-generator` can derive RED metrics directly from spans, avoiding hand-instrumented duplicate metrics.

## Common Mistakes

- Expecting to "search Tempo" the way you'd search Elasticsearch/Jaeger-on-ES for arbitrary tags, without realizing default Tempo is ID-lookup-optimized — TraceQL helps but broad unindexed scans over long time ranges can be slow/expensive.
- Not configuring structured logging with `trace_id`/`span_id` fields, then wondering why Grafana's "trace to logs" button returns nothing — the correlation only works if the log line actually contains the ID to search for.
- Forgetting to configure/enable exemplars on Prometheus/Mimir histograms, losing the fastest metric-to-trace jump path and falling back to manual time-range guessing.
- Running Tempo without a compactor or retention policy configured, letting object storage accumulate ever-growing numbers of small blocks, which slows down querying and inflates storage cost over time.
- Assuming Tempo needs its own dedicated database — the whole point is that it doesn't; provisioning a separate stateful cluster "just in case" defeats the architecture's cost/ops advantage.
