# Day 94 — OpenTelemetry: Collector Architecture, Context Propagation & Sampling

**Phase:** 3 – Observability | **Week:** W16 | **Domain:** Tracing | **Flag:** ⚡ Interview-critical

## Brief

Three things separate a toy OTel setup from a production one: (1) routing telemetry through a **Collector** instead of straight to a backend, (2) understanding **how trace context crosses process/service boundaries** so a single trace stays connected across a microservice call chain, and (3) choosing a **sampling strategy** so you don't bankrupt yourself storing 100% of traces at scale. This file covers all three — and directly answers today's interview question about trace context propagation.

## The OpenTelemetry Collector

The Collector is a standalone, vendor-agnostic process (deployed as a sidecar, a per-host daemonset, or a centralized gateway) that decouples **instrumented applications** from **observability backends**. Instead of every service's SDK exporting directly to Jaeger/Datadog/Honeycomb, every service exports to the Collector, and the Collector fans out from there.

Its pipeline has three stages, wired together in a config file:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch: {}
  memory_limiter:
    check_interval: 1s
    limit_mib: 512
  tail_sampling:
    policies:
      - name: sample-errors
        type: status_code
        status_code: {status_codes: [ERROR]}
      - name: sample-slow
        type: latency
        latency: {threshold_ms: 500}
      - name: sample-rest
        type: probabilistic
        probabilistic: {sampling_percentage: 10}
  attributes:
    actions:
      - key: http.request.header.authorization
        action: delete   # strip sensitive data before export

exporters:
  otlp/jaeger:
    endpoint: jaeger-collector:4317
    tls: {insecure: true}
  prometheus:
    endpoint: 0.0.0.0:8889
  logging:
    verbosity: detailed

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, tail_sampling, batch]
      exporters: [otlp/jaeger, logging]
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [prometheus]
```

- **Receivers** — how telemetry gets *in* (OTLP gRPC/HTTP, but also Jaeger-native, Zipkin, Prometheus scrape, host metrics, etc.). A Collector can ingest from multiple formats simultaneously, which is how you migrate incrementally from legacy Jaeger-client instrumentation to OTel.
- **Processors** — transform telemetry in-flight: batching (throughput), memory limiting (back-pressure so the Collector doesn't OOM under a traffic spike), attribute scrubbing (redact PII/secrets before they ever leave your network), tail sampling (see below), resource detection.
- **Exporters** — where telemetry goes *out* — one or many backends simultaneously. This is the vendor-neutrality payoff: fan the same data out to Jaeger *and* a SaaS backend during a migration, or send traces to Tempo while sending derived metrics to Prometheus.

### Deployment patterns

- **Agent (sidecar/daemonset)** — one Collector instance per host or per pod, close to the application, handling simple batching/enrichment before forwarding onward. Low latency, no single point of failure for a whole cluster, but harder to do global operations like tail sampling correctly (see below).
- **Gateway (centralized)** — a pool of Collector instances behind a load balancer that every agent (or every app directly) forwards to. Centralizes expensive processing (tail sampling needs to see *all* spans of a trace, which usually means routing every span of a given trace to the *same* gateway instance, typically via consistent-hashing load balancing on `trace_id`) and lets you scale/manage the "hard" processing separately from application hosts.
- Production topologies commonly run **both**: agents on every host doing cheap local work (batching, resource tagging), forwarding to a central gateway tier that does tail sampling, scrubbing, and fan-out to backends.

**Why route through a Collector instead of exporting directly from the SDK?** Decoupling: you can change backends, add PII scrubbing, or switch sampling strategy by editing one Collector config — with zero application redeploys. It also protects backends from thundering-herd traffic spikes (the Collector buffers/batches) and centralizes the expensive tail-sampling decision logic instead of duplicating it in every service.

## Trace context propagation (W3C TraceContext) — the interview question

**The question:** *Explain how distributed trace context propagates across service boundaries. What headers are used?*

**The mechanism:** When Service A calls Service B over HTTP, the *only* way B knows "this request belongs to trace X, and it's a child of span Y in A" is if A **injects** that information into the outgoing request, and B **extracts** it from the incoming request. OTel standardizes this using the **W3C Trace Context** specification, via two HTTP headers:

- **`traceparent`** — the mandatory header, format:
  ```
  traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
  ```
  Broken down: `version-trace_id-parent_id-trace_flags`
  - `00` — spec version.
  - `4bf92f3577b34da6a3ce929d0e0e4736` — the 16-byte (32 hex char) `trace_id`, shared by every span in the entire distributed trace, generated once by the very first service that starts the trace.
  - `00f067aa0ba902b7` — the 8-byte `span_id` of the **calling** span in Service A — this becomes the `parent_span_id` for whatever span Service B creates to represent handling this request.
  - `01` — trace flags (bit 0 = "sampled": tells downstream services whether this trace was sampled-in upstream, so they don't have to re-decide — see sampling below).
- **`tracestate`** — an optional, vendor-specific extension header for additional cross-vendor state, format `vendorname1=opaqueValue1,vendorname2=opaqueValue2`. Lets multiple observability vendors/systems pass their own context through the same request without clobbering each other.

**What actually happens, step by step:**
1. Service A's HTTP client instrumentation (auto or manual) reads the current active span's `trace_id` and `span_id` and injects a `traceparent` header into the outgoing request before it goes over the wire.
2. Service B's HTTP server instrumentation, on receiving the request, **extracts** the `traceparent` header, and instead of starting a brand-new trace, it creates its span as a child using the extracted `trace_id` and sets `parent_span_id` to the extracted `span_id`.
3. This repeats at every hop — HTTP, but also gRPC metadata, Kafka message headers, SQS message attributes, etc. — anywhere a message crosses a process boundary, propagation needs an explicit inject/extract step. This is precisely why message-queue-based architectures need extra instrumentation care: unlike a synchronous HTTP call, you must manually propagate context into the message payload/headers on publish and extract it on consume, or the trace breaks into two disconnected traces at the queue boundary.
4. The result: every span across every service, for one logical request, shares one `trace_id`, and the `parent_span_id` chain reconstructs the exact call graph — which is what renders as the waterfall/flame graph in Jaeger/Tempo.

**Why W3C standardization matters:** before this, every vendor (B3 headers from Zipkin, custom headers from Datadog/AWS X-Ray) had its own propagation format, so a request crossing services instrumented by different vendors' SDKs would silently break the trace at that boundary. W3C TraceContext is now a web standard (not just an OTel thing), so any compliant library — regardless of vendor — can propagate context correctly.

## Sampling strategies: head vs. tail

Sampling exists because tracing *every* request at scale is often prohibitively expensive (storage + backend ingestion cost), so you decide which traces to keep.

- **Head-based sampling** — the sampling decision is made **at the start** of the trace (usually by the first service), before anything is known about how the request will turn out (fast/slow, success/error). Cheap and simple — e.g., "sample 10% of traces, chosen by a probabilistic hash of the trace_id" — and the decision (encoded in the `traceparent` flags byte) is passed downstream so every service honors the same decision, keeping the trace complete. **Downside:** you can't sample based on outcome — you might drop the one trace that would've shown you the rare, painful error, and keep 999 boring successful traces instead.
- **Tail-based sampling** — the decision is made **after** the entire trace is complete (all spans collected), so you can sample based on actual outcome: "keep 100% of traces with errors, 100% of traces slower than 500ms, and only 10% of everything else" (exactly the `tail_sampling` processor config shown above). This catches the traces you actually care about for debugging. **Downside:** requires buffering the *entire* trace somewhere (usually the Collector gateway tier) until it's complete before deciding — more memory, more complexity, and requires all spans of one trace to reach the same Collector instance (hence consistent-hashing load balancers keyed on `trace_id` in gateway deployments).

**Practical guidance:** head sampling is simpler and cheaper at very large scale where you mostly care about statistical trends; tail sampling is what you want when debugging specific failures/latency outliers is the priority — which is most SRE/on-call use cases. Many production setups combine both: a cheap head-based pre-filter to cut obvious low-value volume, then tail-based rules on what remains to guarantee errors/slow requests are always kept.

## Points to Remember

- Collector pipeline = receivers (in) → processors (transform/filter/sample) → exporters (out); one config, multiple simultaneous inputs/outputs.
- Agent (per-host) vs. gateway (centralized) deployment — tail sampling requires all of a trace's spans to land on the same instance, which is why it's usually done at a gateway tier with consistent-hash load balancing on `trace_id`.
- `traceparent` header format: `version-trace_id-parent_id-trace_flags`; `trace_id` is shared across the whole distributed trace, `parent_id` changes at every hop.
- Context propagation across message queues (Kafka/SQS/etc.) is NOT automatic like HTTP — it requires explicit inject-on-publish / extract-on-consume, or the trace splits in two at the queue.
- Head sampling decides upfront (cheap, can't use outcome); tail sampling decides after seeing the whole trace (can guarantee errors/slow requests are kept, costs more to buffer).

## Common Mistakes

- Exporting directly from every service's SDK straight to the backend instead of through a Collector — works at small scale, but makes backend migration, PII scrubbing, and tail sampling far harder to retrofit later.
- Assuming trace context propagates "automatically" across a message queue the way it does over HTTP — forgetting to inject `traceparent` into Kafka headers/SQS message attributes on publish silently breaks the trace into two disconnected traces at the queue boundary.
- Running tail sampling on a fleet of independent agent-mode Collectors without a consistent-hashing load balancer in front — spans of the same trace land on different Collector instances, none of which sees the "whole" trace, so the sampling decision is nonsensical.
- Confusing the "sampled" flag bit in `traceparent` with "this trace is definitely stored" — the flag communicates the upstream head-sampling decision; a tail-sampling processor further downstream can still decide to drop it.
- Using probabilistic-only (head) sampling and then being surprised that the one critical production error never shows up in the tracing backend because it happened to fall outside the sampled 10%.
