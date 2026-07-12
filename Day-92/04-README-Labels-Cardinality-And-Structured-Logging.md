# Day 92 — Loki & Log Aggregation: Log Labels, Cardinality, Structured Logging & Grafana Correlation

**Phase:** 3 – Observability | **Week:** W15 | **Domain:** Logging | **Flag:** ⚡ Interview-critical

## Brief

Day 91 covered cardinality as a metrics problem; it's just as real — arguably more commonly mishandled — in Loki, because the instinct to "just add a label so I can filter on it" is even stronger for logs than for metrics. This file closes the day by tying together label discipline, structured (JSON) logging practices, and how everything from Days 89–92 (Prometheus, Grafana, Loki) fits together into one correlated debugging workflow.

## Log labels and cardinality in Loki

Recall from file 1: Loki indexes **only labels**, and each unique label-value combination creates a distinct **stream** with its own chunk storage. This means Loki's cardinality problem has the same shape as Prometheus's, with one Loki-specific twist:

- **Every unique label combination is a new stream** — exactly like every unique label combination in Prometheus is a new time series. A label like `pod` (which changes on every deployment/restart) is already borderline; a label like `trace_id`, `request_id`, or `user_id` would be catastrophic, creating a practically infinite number of tiny streams.
- **Loki-specific twist**: because each stream is stored as its own compressed chunk, high cardinality doesn't just bloat an index — it also produces a huge number of **small, inefficiently-compressed chunks** (compression works better over longer runs of similar data within one stream), directly hurting both storage efficiency and query performance in a way that compounds the usual "too many index entries" problem.
- **The fix, mirroring Day 91's metrics advice**: keep labels to genuinely low-cardinality, structural dimensions — `namespace`, `app`/`job`, `container`, `level`, maybe `environment` — and put anything high-cardinality (`trace_id`, `user_id`, `request_id`, a specific error message) **into the log line content itself**, queryable via LogQL's `| json` / `| logfmt` parsing at query time rather than as an indexed label. This is precisely why Loki's "index only a few labels, grep the rest" architecture exists — it's a deliberate design constraint, not an oversight, and working with it (not against it) is the whole skill.

## Structured logging (JSON) best practices

Unstructured logs (`"2026-01-01 12:00:00 ERROR failed to process order 12345 for user bob: timeout"`) require regex gymnastics to extract anything reliably. **Structured logging** — emitting each log line as JSON (or logfmt) with consistent field names — is what makes LogQL's `| json` parser (and equivalent tooling in ELK/OpenSearch) actually useful:

```json
{"timestamp":"2026-01-01T12:00:00Z","level":"error","service":"checkout","order_id":"12345","user_id":"bob","message":"failed to process order","error":"timeout","trace_id":"a1b2c3d4"}
```

Best practices that matter in practice, not just in theory:
- **Consistent field names across services** — if one service logs `user_id` and another logs `userId` or `uid` for the same concept, every cross-service LogQL query has to account for all three, or (more realistically) nobody bothers and cross-service correlation quietly doesn't happen. Establish a shared logging schema/library across your organization early.
- **Always include a correlation ID** (`trace_id` at minimum, ideally propagated via OpenTelemetry context) — this is the field that makes the Day 90 "logs → traces" Grafana correlation actually work; without it in every log line, there's no mechanism to jump from a log line to its trace.
- **Log levels as an actual field (`level`), not just embedded text** — lets you filter `{app="api"} | json | level="error"` cleanly instead of hoping the word "error" never appears in a non-error message body (it will — e.g., "successfully handled error-recovery flow" is a real false-positive risk with naive text `|=` filtering).
- **Don't put high-cardinality values in what becomes a Loki *label*** — put them in the JSON body instead, per the cardinality discussion above; the JSON body is exactly where `order_id`, `user_id`, `trace_id` belong.
- **Avoid multi-line, unstructured stack traces breaking your one-line-per-log-entry assumption** — configure your logging library to either serialize the whole stack trace as a single JSON string field or ensure your log shipper's multiline-detection is correctly configured, or a single exception can appear to Loki as dozens of separate malformed "log lines."

## Correlating logs with metrics and traces in Grafana

This ties directly back to Day 90's cross-source correlation discussion — now with the log side filled in:

- **From a Prometheus dashboard panel showing a spike** → split-screen or pivot into Grafana Explore with a Loki query scoped to the same time range and the same `namespace`/`pod` labels, using LogQL filters (`|= "error"` or `| json | level="error"`) to pull the exact error lines from that window.
- **From a Loki log line** with a `trace_id` field → a configured `derivedFields` regex match turns it into a clickable link straight into the matching Tempo trace (exact mechanism covered in Day 90, file 3) — this only works because the log line actually contains `trace_id` as a structured field, reinforcing why structured logging with correlation IDs isn't optional if you want this workflow.
- **Grafana Explore's "Logs" view** lets you visually scan a histogram of matching log volume over time (a bar chart above the raw log lines) — clicking a bar zooms the time range to that spike, a fast way to visually locate "when did the error volume actually start" before reading a single line.

The reason this whole day matters together, not as isolated facts: an actual incident workflow looks like *"Grafana dashboard shows p99 latency spike (Prometheus) → pivot to Loki logs scoped to the same service/time window, filtered to `level=error` → find a `trace_id` in the matching log lines → click through to the full distributed trace in Tempo to see exactly which downstream call was slow."* Each piece (metrics, logs, correlation IDs, structured logging, Grafana wiring) has to be in place for that workflow to actually work at 3am — this is why "compare Loki and Elasticsearch" interview questions often expect you to also talk about label discipline and structured logging, not just storage architecture in isolation.

## Points to Remember

- Every unique Loki label combination is a separate stream/chunk — high-cardinality labels don't just bloat an index like in Prometheus, they also fragment compression across many small chunks, compounding the performance cost.
- Keep Loki labels to genuinely low-cardinality structural dimensions (namespace, app, container, level); put high-cardinality identifiers (trace_id, user_id, order_id) in the JSON log body, queryable via `| json` at query time instead.
- Structured (JSON/logfmt) logging with a consistent schema across services is what makes both `| json` parsing and cross-service correlation actually work — inconsistent field naming quietly breaks organization-wide log queries.
- A `trace_id` field in every log line (via OpenTelemetry context propagation) is the specific mechanism that enables one-click log→trace correlation in Grafana.
- The real value of this whole day is the end-to-end workflow — metric spike → correlated logs → correlated trace — not any one tool in isolation.

## Common Mistakes

- Promoting a high-cardinality field (user_id, request_id, a raw error message) to an actual Loki *label* instead of leaving it in the JSON body — this is the Loki-specific version of the exact same mistake covered for Prometheus on Day 91, and it's just as damaging here.
- Inconsistent field naming across services' structured logs (`user_id` vs `userId` vs `uid`), silently breaking any query or dashboard meant to work across more than one service.
- Logging free-text messages with the word "error" embedded even for non-error-level log lines, then filtering with a naive `|= "error"` text match instead of `| json | level="error"` — produces false positives that make incident triage slower, not faster.
- Not propagating a correlation/trace ID through async boundaries (message queues, background jobs) — logs on the "other side" of a queue lose the ability to be correlated back to the original request's trace, breaking exactly the workflow this file describes at the point where it matters most (async failures are often the hardest to debug already).
- Letting multi-line stack traces ship as multiple separate unstructured log lines/streams instead of configuring proper multiline handling in the log shipper — fragments a single exception into dozens of misleading "log entries" in Loki.
