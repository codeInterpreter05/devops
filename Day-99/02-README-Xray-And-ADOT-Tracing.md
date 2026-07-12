# Day 99 — AWS-native Observability II: X-Ray & ADOT Tracing

**Phase:** 3 – Observability | **Week:** W17 | **Domain:** Observability | **Flag:** ⚡ Interview-critical

## Brief

Metrics tell you *that* something is slow; logs tell you *what* happened on one host; only distributed tracing tells you *where in a chain of 12 microservices* the latency or error actually originated. X-Ray is AWS's native tracing backend, and ADOT is AWS's supported packaging of the vendor-neutral OpenTelemetry project. Understanding both — and why the industry is moving toward the second — is a strong signal of someone who's thought about observability architecture, not just clicked around a console.

## X-Ray fundamentals

X-Ray is a distributed tracing **backend and visualization layer**. The core primitives:

- **Segment** — a record of work done by one service for one request (start time, end time, HTTP status, errors/faults, annotations/metadata).
- **Subsegment** — a nested unit of work inside a segment (e.g., a specific DynamoDB call or an outbound HTTP request made while handling the request).
- **Trace** — all segments/subsegments sharing the same **trace ID**, stitched together into one end-to-end view of a request across every service it touched.
- **Trace ID propagation** — carried via the `X-Amzn-Trace-Id` HTTP header (`Root=1-...;Parent=...;Sampled=1`). Every service in the call chain must read this header, add its own segment, and forward it downstream — if one hop in the chain doesn't propagate it, the trace breaks into two disconnected fragments.

**How segments get to X-Ray:** the **X-Ray daemon** (or the equivalent sidecar in ECS/EKS) listens on UDP port 2000 on the local host, batches segments emitted by the X-Ray SDK in your application, and forwards them to the X-Ray API over HTTPS. This UDP-then-batch design is deliberate — it means instrumenting your app never blocks or slows down the request path waiting on a network call to AWS; worst case, the daemon is unreachable and segments are dropped, not the request.

**Sampling** matters at any real scale: X-Ray's default sampling rule captures the first request per second, plus 5% of any requests beyond that, per service. This exists because tracing every single request at high QPS would be both enormously expensive (X-Ray bills per trace recorded) and mostly redundant — a service handling 10,000 identical health checks a minute doesn't need 10,000 traces to tell you it's healthy. Custom sampling rules let you raise the rate for a specific route (e.g., checkout) while keeping it low for high-volume, low-value paths (e.g., `/healthz`).

The **Service Map** is X-Ray's headline UI: a node-and-edge graph of every service that participated in traced requests, colored by error/fault rate, with median and p99 latency per edge — the fastest way to answer "which hop in this call chain is actually slow" without reading a single log line.

## ADOT — AWS Distro for OpenTelemetry

The problem with instrumenting directly against the X-Ray SDK: your application code (and its trace data format) becomes tied to one vendor. If you ever want to send traces to a different backend (Jaeger, Honeycomb, Datadog, a different cloud), you re-instrument from scratch.

**ADOT** solves this by being AWS's supported, tested distribution of the **OpenTelemetry (OTel)** SDKs and the **OpenTelemetry Collector** — the CNCF-standard, vendor-neutral instrumentation layer. The workflow:

1. Your application is instrumented with vanilla OpenTelemetry SDKs (auto-instrumentation agents exist for Java, Python, .NET, Node.js — often zero code changes).
2. The app emits traces/metrics over the **OTLP** protocol (OpenTelemetry Line Protocol, gRPC or HTTP) to a local **ADOT Collector**, typically run as a sidecar container or a per-node DaemonSet in EKS.
3. The ADOT Collector's **exporters** decide where data goes — it can export traces to X-Ray, metrics to CloudWatch (via the EMF — Embedded Metric Format — exporter), or simultaneously to a third-party backend, all from the same instrumentation.

```yaml
# adot-collector-config.yaml (ConfigMap consumed by the ADOT Collector)
receivers:
  otlp:
    protocols:
      grpc:
      http:
exporters:
  awsxray:
    region: us-east-1
  awsemf:
    region: us-east-1
    namespace: MyApp
service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [awsxray]
    metrics:
      receivers: [otlp]
      exporters: [awsemf]
```

**Why this is the recommended modern path**: it decouples *instrumentation* (OTel SDK calls in your code, a portable standard) from *backend choice* (an exporter config, changeable without touching application code). If your org later migrates off X-Ray to Grafana Tempo or Datadog APM, you swap an exporter in a collector config — you do not re-instrument every service. This is exactly the kind of architectural reasoning interviewers want to hear: "we chose OTel/ADOT over the raw X-Ray SDK specifically to avoid vendor lock-in on the instrumentation layer, even though we're exporting to X-Ray today."

## Points to Remember

- A trace is only as complete as its weakest propagation hop — if any service in the chain doesn't forward `X-Amzn-Trace-Id`, the trace fragments into disconnected pieces with no visible link between them.
- X-Ray's default sampling (1/sec + 5% overflow, per service) exists to control both cost and noise; tune sampling rules per-route based on traffic value, not blanket-increase the global rate.
- The X-Ray daemon listens on UDP 2000 and batches asynchronously — instrumentation is fire-and-forget from the app's perspective, so a daemon outage degrades observability, not the request path.
- ADOT = AWS's supported OpenTelemetry distribution; the value proposition is vendor-neutral instrumentation with swappable exporters (X-Ray, CloudWatch EMF, or third-party backends) from the same SDK calls.
- The Service Map is the fastest way to localize latency/error hot-spots across a multi-service call chain — always check it before diving into individual logs.

## Common Mistakes

- Instrumenting only some services in a call chain with X-Ray, then being confused why traces "stop" partway through — the gap is a service that never forwards or creates the trace header.
- Leaving default sampling untouched on a critical, low-traffic, high-value route (e.g., checkout) and then having almost no trace data during an actual incident because the request volume never crossed the 1/sec threshold.
- Choosing the raw X-Ray SDK for new instrumentation in 2024+ without considering ADOT/OpenTelemetry, then having to re-instrument everything later when a multi-cloud or multi-backend requirement appears.
- Assuming the ADOT Collector requires no capacity planning — under sustained high trace/metric volume, an under-resourced Collector pod can itself become a bottleneck or drop data.
- Forgetting that X-Ray bills per trace recorded (not per span) — teams sometimes assume "sampling saves nothing" and disable it, then get an unpleasant bill surprise under load spikes.
