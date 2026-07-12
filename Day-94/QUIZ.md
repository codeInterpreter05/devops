# Day 94 — Quiz: OpenTelemetry

Try to answer without looking at your notes. Answers are at the bottom.

1. What is OpenTelemetry, precisely? Name two things it explicitly is NOT.
2. Name the three OTel signals and, for each, the one question it's best suited to answer.
3. What are the three OTel metric instrument types, and what's the difference between a Counter and a Gauge?
4. What's the mechanical difference between auto-instrumentation and manual instrumentation in the Python SDK? Give one example of when you'd need manual instrumentation even with auto-instrumentation enabled.
5. Why must you call `span.record_exception()` and `set_status(ERROR)` explicitly when you catch an exception inside a span?
6. What does the OTel Collector do that justifies adding an extra hop instead of exporting straight from the SDK to a backend?
7. What are the three stages of a Collector pipeline, and what does each do?
8. What's the difference between an "agent" and a "gateway" Collector deployment? Why does tail sampling specifically need a gateway-style deployment with consistent hashing?
9. **Interview question:** Explain how distributed trace context propagates across service boundaries. What headers are used?
10. Why doesn't trace context propagate "automatically" across a Kafka or SQS message the way it does over a synchronous HTTP call?
11. What's the core tradeoff between head-based and tail-based sampling?
12. Why is `BatchSpanProcessor` the production default over `SimpleSpanProcessor`?

---

## Answers

1. OpenTelemetry is a vendor-neutral specification + set of language SDKs + a Collector + the OTLP wire protocol for generating and transporting telemetry (traces, metrics, logs). It is explicitly **not** a storage backend and **not** a visualization/UI tool — you still need something like Jaeger, Tempo, Prometheus+Grafana, or a SaaS vendor to store and view the data.
2. Traces — "why was this specific request slow/erroring" (request-scoped path). Metrics — "what's the overall trend/rate across all requests" (aggregated). Logs — "what exactly happened at this moment" (detailed discrete events).
3. Counter (monotonically increasing, e.g. total requests), Gauge (can go up or down, e.g. queue depth), Histogram (bucketed distribution enabling percentile computation, e.g. request duration). Counter vs Gauge: a Counter can only ever increase (or reset to zero on restart); a Gauge reflects a point-in-time value that can rise or fall freely.
4. Auto-instrumentation patches third-party library code (bytecode/monkey-patch) at runtime via `opentelemetry-instrument`/`opentelemetry-bootstrap`, requiring zero application code changes — it only knows about library-level operations (HTTP handling, DB calls). Manual instrumentation is explicit `tracer.start_as_current_span(...)` code you write. Example needing manual: tagging a span with `discount.code` or timing a business-specific "apply_discount" step — auto-instrumentation has no way to know this logic exists.
5. Because a span's status defaults to unset/OK — if you catch an exception and don't explicitly mark the span as ERROR, the trace UI will show a fast, "successful"-looking span even though the operation actually failed, hiding the real problem from anyone debugging via traces alone.
6. The Collector decouples instrumented apps from backends: you can change backends, add PII/secret scrubbing, or add tail sampling by editing one Collector config with zero application redeploys. It also buffers/batches to protect backends from traffic spikes and centralizes expensive processing (like tail sampling) instead of duplicating logic in every service.
7. Receivers (get telemetry in — OTLP, Jaeger, Zipkin, Prometheus scrape, etc.), Processors (transform in-flight — batching, memory limiting, attribute scrubbing, tail sampling), Exporters (send telemetry out to one or more backends).
8. Agent = one Collector per host/pod, close to the app, doing lightweight local work (batching, tagging). Gateway = a centralized pool of Collector instances that agents (or apps) forward to, used for heavier processing. Tail sampling needs to see every span belonging to a trace before deciding whether to keep it, which means all spans of one trace must land on the *same* Collector instance — achieved by consistent-hash load balancing keyed on `trace_id` in front of a gateway tier; a fleet of independent agents can't guarantee this.
9. OTel uses the W3C TraceContext standard, propagated via the `traceparent` HTTP header (format: `version-trace_id-parent_id-trace_flags`) and optionally `tracestate`. The calling service's client instrumentation injects `traceparent` into the outgoing request using the current trace_id and its own span_id; the receiving service's server instrumentation extracts it and creates its span as a child of that trace_id/span_id instead of starting a new trace. This repeats at every hop, so every span across every service shares one trace_id and forms a connected parent/child tree.
10. HTTP instrumentation automatically injects/extracts `traceparent` as part of standard request/response handling. Message queues have no equivalent standard request/response cycle — the publishing service must manually inject `traceparent` into the message's headers/attributes, and the consuming service must manually extract it; if this step is skipped, the trace silently splits into two disconnected traces at the queue boundary.
11. Head-based sampling decides at trace start (cheap, propagated via the sampled flag, but can't factor in outcome — you might drop the one erroring/slow trace you actually needed). Tail-based sampling decides after the full trace is collected (can guarantee errors/slow traces are always kept), but requires buffering the entire trace and routing all its spans to one Collector instance, which costs more memory and complexity.
12. `BatchSpanProcessor` buffers finished spans and exports them in batches on a timer/size threshold, avoiding a network call per span. `SimpleSpanProcessor` exports synchronously, one span at a time, adding real latency to the request's hot path and tanking throughput under load — acceptable only for local debugging/tests, never production.
