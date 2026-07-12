# Day 94 — OpenTelemetry: Overview & the Three Signals

**Phase:** 3 – Observability | **Week:** W16 | **Domain:** Tracing | **Flag:** ⚡ Interview-critical

## Brief

Observability used to mean three separate tools (a metrics system, a log aggregator, and — if you were lucky — a tracing system) glued together with duct tape and vendor lock-in. OpenTelemetry (OTel) is the CNCF project that unifies how you *generate and transport* telemetry — traces, metrics, and logs — behind a single, vendor-neutral API and wire protocol (OTLP). It matters for real work because almost every modern observability stack (Datadog, New Relic, Honeycomb, Grafana Cloud, Splunk, self-hosted Prometheus/Tempo/Loki) now speaks OTLP natively — instrument once with OTel, and you can point the same telemetry at any backend without re-instrumenting your code. It matters for interviews because "how would you add observability to a service without locking yourself into one vendor" is now a standard senior/SRE question, and OTel is the expected answer.

This day is split into three focused files:

1. **This file** — what OTel is, the three signals (traces/metrics/logs), and why a unified pipeline matters.
2. **[02-README-Instrumentation-Auto-Vs-Manual.md](02-README-Instrumentation-Auto-Vs-Manual.md)** — auto-instrumentation vs. manual instrumentation, and the Python SDK in practice.
3. **[03-README-Collector-Propagation-Sampling.md](03-README-Collector-Propagation-Sampling.md)** — Collector architecture, W3C trace context propagation, and head vs. tail sampling.

## What OpenTelemetry actually is

OpenTelemetry is **not** a backend/storage system — it stores nothing and has no UI. It is:

- A **specification** for what a trace, metric, and log record must look like (the data model).
- A set of **language SDKs** (Python, Go, Java, JS, .NET, etc.) that implement that spec so your application code can emit telemetry.
- A wire protocol, **OTLP** (OpenTelemetry Protocol, gRPC or HTTP/protobuf), for shipping that telemetry somewhere.
- The **Collector**, a standalone binary that receives, processes, and exports telemetry (deep dive in file 3).

Think of OTel as the "USB-C of observability": one instrumentation standard, many possible destinations. Before OTel, you'd instrument with a vendor SDK (e.g., Datadog's `ddtrace`) and switching backends meant re-instrumenting every service. With OTel, you swap the *exporter* configuration, not the application code.

OTel was formed by merging two earlier CNCF projects — **OpenTracing** (traces) and **OpenCensus** (traces + metrics, from Google) — in 2019, which is why it inherited a broad, sometimes verbose API surface: it had to be a superset of both.

## The three signals

| Signal | What it captures | Best for |
|---|---|---|
| **Traces** | The path a single request takes across services/functions, as a tree of timed **spans** | "Why was *this specific* request slow/erroring?" — request-scoped debugging |
| **Metrics** | Aggregated numeric measurements over time (counters, gauges, histograms) | "What's the overall error rate / p99 latency / throughput across all requests?" — trends, alerting, capacity |
| **Logs** | Discrete, timestamped event records, usually free-text or structured (JSON) | "What exactly happened, in detail, at this moment?" — root-cause detail, audit trail |

These are often called the **three pillars of observability**, and the real power isn't any one of them — it's **correlating** them. A metric tells you *something* is wrong (error rate spiked); a trace tells you *which* request path and *which* downstream service is responsible; a log tells you the *exact* exception/stack trace/payload at that point. OTel's data model deliberately ties these together: every span, log record, and (optionally) metric data point carries a `trace_id`/`span_id`, so a backend like Grafana or Honeycomb can jump from a spike on a dashboard straight to the individual failing traces and their associated log lines, with no manual correlation.

### Anatomy of a trace

- A **trace** represents one end-to-end request/transaction, identified by a single `trace_id`.
- A trace is composed of one or more **spans** — each span is a named, timed unit of work (e.g., "HTTP GET /checkout", "DB query: SELECT * FROM orders", "call to payment-service"). Each span has a `span_id`, a `parent_span_id` (forming a tree), start/end timestamps, a status (OK/Error), and **attributes** (key-value metadata like `http.status_code`, `db.statement`) and **events** (timestamped points inside a span, e.g., an exception being thrown).
- The span tree *is* the trace — visualized as a waterfall/flame graph, it shows exactly which hop in a request ate the most time.

### Anatomy of a metric

OTel metrics come in three primary instrument types:
- **Counter** — monotonically increasing value (`http_requests_total`). Never decreases.
- **Gauge** — a value that can go up or down at any time (`queue_depth`, `memory_used_bytes`).
- **Histogram** — buckets observed values to let you compute percentiles/distributions after the fact (`http_request_duration_seconds`). This is what lets you compute p50/p95/p99 latency without storing every single raw sample.

### Anatomy of a log

An OTel **LogRecord** has a timestamp, severity, body (the message, ideally structured), and — critically — **optional trace/span IDs** injected automatically when the log call happens inside an active span's context. This is the mechanism that makes "click a trace, jump to its logs" possible without manually threading a request ID through every log line yourself.

## Why this matters operationally

- **Vendor neutrality is a cost/risk decision, not just a technical one.** A company that instruments with a proprietary SDK is paying an implicit "instrumentation tax" if it ever needs to migrate observability vendors (a multi-quarter re-instrumentation project). OTel converts that into a config change.
- **Correlated signals cut MTTR (mean time to resolution) dramatically.** Without trace-to-log correlation, an on-call engineer manually greps logs by timestamp and guesses which log lines belong to the slow request. With `trace_id` correlation, it's one click.
- **The three signals answer different questions at different zoom levels** — metrics for "is there a problem" (cheap, always-on, aggregated), traces for "where is the problem" (request-scoped, sampled), logs for "what exactly happened" (detailed, often the most expensive to store at scale). Good observability design deliberately uses each for what it's cheap and good at, rather than trying to make one signal do everything (e.g., don't try to debug root cause purely from metrics, and don't log every field of every request just because logs are easy to add).

## Points to Remember

- OpenTelemetry is a specification + SDKs + Collector + OTLP protocol. It is **not** a storage backend or a UI — you still need Jaeger/Tempo/Prometheus/Grafana/a vendor to visualize and store the data.
- The three signals: traces (request path), metrics (aggregated trends), logs (detailed events) — each optimized for a different question, correlated via `trace_id`/`span_id`.
- OTel merged OpenTracing + OpenCensus in 2019 — that heritage explains some of its API breadth.
- Counter (monotonic), Gauge (up/down), Histogram (distribution/percentiles) are the three core metric instrument types.
- A span's parent/child relationships form the trace tree; that tree is what renders as a flame graph / waterfall diagram.

## Common Mistakes

- Calling OpenTelemetry "a monitoring tool" in an interview — it's the instrumentation/transport layer; you still need a backend (this distinction is exactly what interviewers probe for).
- Treating logs, metrics, and traces as three unrelated systems to bolt on separately, instead of designing for correlation (structured logs with `trace_id` injected) from day one — retrofitting correlation later is painful.
- Trying to compute percentiles (p99) by averaging pre-aggregated averages from multiple hosts — you need histograms (or raw samples) aggregated centrally; averages of averages produce mathematically meaningless percentiles.
- Assuming "adding OTel" is free performance-wise — unsampled, verbose tracing (or unbounded log volume) can add meaningful latency and cost at scale; sampling and attribute cardinality control (file 3) exist precisely to manage this.
- Confusing "vendor-neutral instrumentation" with "vendor-neutral everything" — you can switch *exporters* freely, but dashboards/alerts built in a specific backend's query language still need to be rebuilt if you migrate backends.
