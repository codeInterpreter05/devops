# Day 92 — Loki & Log Aggregation: Loki vs. ELK Architecture

**Phase:** 3 – Observability | **Week:** W15 | **Domain:** Logging | **Flag:** ⚡ Interview-critical

## Brief

Metrics (Days 89–91) tell you *that* something is wrong; logs tell you *what actually happened*. This day shifts the whole phase from metrics to logs, and the very first architectural decision you'll be asked about in an interview is "Loki or ELK/OpenSearch" — because the two systems make almost opposite tradeoffs about what gets indexed, and picking wrong means either an unmanageable cost bill or a system that can't answer the queries your team actually needs.

This day is split into four focused files:

1. **This file** — Loki vs. ELK architecture comparison.
2. **[02-README-LogQL-Basics.md](02-README-LogQL-Basics.md)** — LogQL, Loki's PromQL-like query language.
3. **[03-README-Log-Shippers-Promtail-FluentBit-Vector.md](03-README-Log-Shippers-Promtail-FluentBit-Vector.md)** — Promtail vs. Fluent Bit vs. Vector.
4. **[04-README-Labels-Cardinality-And-Structured-Logging.md](04-README-Labels-Cardinality-And-Structured-Logging.md)** — log labels/cardinality, structured (JSON) logging, and Grafana correlation.

## The core architectural difference

**ELK (Elasticsearch/Logstash/Kibana) and its OpenSearch fork** work by **indexing the full text of every log line** at ingest time — every field, and often every token in free-text fields, gets written into an inverted index so it can be searched and filtered arbitrarily fast, on any field, after the fact. This is powerful (near-instant full-text search across any field) but expensive: the index is frequently several times the size of the raw log data itself, and that indexing work happens on every single log line whether or not anyone ever queries it.

**Loki** takes the opposite bet: it **only indexes a small set of labels** (like Prometheus does for metrics) — typically `namespace`, `pod`, `app`, `container`, `level` — and stores the **raw, unindexed log content** compressed in chunks, in cheap object storage (S3/GCS/etc.). A LogQL query first uses the label index to quickly narrow down to the relevant *streams* (label combinations), then does a **grep-style scan** over the actual compressed log content within those streams at query time.

## Why this tradeoff exists, and what it costs each way

| | ELK / OpenSearch | Loki |
|---|---|---|
| **Ingest cost** | Higher — full-text indexing on every line at write time | Lower — only labels are indexed; log body is compressed and stored, not indexed |
| **Storage cost** | Higher — inverted index is often 2–3x+ the raw log size | Lower — compressed chunks close to raw log size, no full-text index overhead |
| **Query speed on indexed/label fields** | Fast | Fast (label index narrows to relevant streams immediately) |
| **Query speed on arbitrary free-text search** | Fast (that's what the index is for) | Slower — has to scan chunk content within matching streams; still fast in practice because streams are pre-narrowed by labels, but genuinely a linear scan over what's left |
| **Operational model** | Closer to a full-text search engine (shares Elasticsearch's operational complexity: shards, replicas, JVM heap tuning) | Closer to Prometheus's model (deliberately designed to feel familiar to teams already running Prometheus) |
| **Best fit** | Rich ad-hoc search across arbitrary unstructured fields, security/audit search, complex aggregations | High-volume Kubernetes container logs where you already know your label dimensions (namespace/pod/app) and mostly filter-then-grep rather than search arbitrary fields |

**The one-sentence framing that lands well in an interview**: "ELK indexes everything so you can search anything, at real infrastructure cost; Loki indexes only labels and treats the log body like Prometheus treats metric samples — cheap to store, and fast as long as your queries start from a label filter, which in a Kubernetes-labeled world they usually do."

## Why Loki was built to mirror Prometheus specifically

Loki's design (from Grafana Labs, the same company behind Grafana) intentionally reuses ideas from Prometheus: labels as the primary index, a similar multi-tenant/horizontally-scalable architecture, and — as covered in the next file — a query language (LogQL) deliberately modeled on PromQL. The stated goal is that a team already comfortable operating Prometheus (service discovery patterns, label-based thinking, similar storage/compaction concerns) can pick up Loki with much less new conceptual overhead than adopting a fundamentally different system like Elasticsearch. This is also why Loki integrates so tightly into Grafana as "the logs equivalent of Prometheus" rather than being a bolt-on separate product.

## Points to Remember

- ELK/OpenSearch indexes full log content at ingest (expensive, but supports fast arbitrary free-text search on any field). Loki indexes only labels (cheap) and treats log bodies as unindexed, compressed, grep-at-query-time content.
- Loki's storage backend is typically object storage (S3/GCS) for chunks, mirroring how Thanos offloads Prometheus data — same cost-efficiency motivation, applied to logs.
- Loki queries are fast *because* labels narrow the search to specific streams first — a Loki query with no label filter (or only a highly broad one) degenerates into a slow full scan.
- Loki was deliberately designed to mirror Prometheus's operational and conceptual model, which is why teams already running Prometheus tend to adopt it with less friction than adopting Elasticsearch/OpenSearch from scratch.
- Neither is "always better" — it's a real architectural tradeoff between ingest/storage cost and query flexibility, and the right choice depends on whether your team's actual query patterns are label-filtered-then-grep or genuinely ad-hoc full-text search.

## Common Mistakes

- Treating Loki as "Elasticsearch but cheaper" without understanding *why* it's cheaper — teams that try to run genuinely unstructured, label-free, arbitrary full-text search workloads against Loki find it much slower than they expected, because that's exactly the workload Loki deliberately doesn't optimize for.
- Adding too many labels to Loki streams to "make search easier," accidentally recreating the exact high-cardinality-index problem Loki was designed to avoid (covered in file 4) — the temptation to treat Loki labels like Elasticsearch fields is a common instinct that backfires.
- Choosing ELK/OpenSearch purely for its more powerful query capabilities without budgeting for the real infrastructure cost (JVM heap sizing, shard management, index storage multiplier) that comes with full-text indexing at scale.
- Assuming "Loki vs ELK" is purely a cost decision — ignoring genuine capability differences like OpenSearch's stronger support for complex nested aggregations, alerting on arbitrary field combinations, and mature security/RBAC tooling that some compliance-heavy use cases genuinely need.
- Not accounting for the fact that a Loki query without any label selector (or an extremely broad one like `{namespace=~".+"}`) can still be very slow — the label index helps only as much as your labels actually narrow the search space.
