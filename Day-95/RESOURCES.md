# Day 95 — Resources: Jaeger & Tempo

## Primary (assigned)

- **Grafana Tempo documentation** (grafana.com/docs/tempo) — free, the assigned starting point for this day. Covers Tempo's architecture, deployment, and TraceQL authoritatively, including the metrics-generator and trace-to-logs correlation setup used in today's lab.

## Deepen your understanding

- **Jaeger documentation — Architecture** (jaegertracing.io/docs) — the official breakdown of collector/query/UI/storage components and the supported storage backends.
- **"Mastering Distributed Tracing"** (Yuri Shkuro, Jaeger's original creator at Uber) — the definitive book on tracing internals, sampling, and Jaeger's design decisions, written by the person who built it.
- **Grafana Labs blog — "Tempo: A New Way to Scale Distributed Tracing"** — explains the object-storage-first design philosophy shared with Loki, and why Tempo deliberately skips a full secondary index.
- **TraceQL documentation** (grafana.com/docs/tempo/latest/traceql) — full query language reference with syntax for filtering by span attributes, duration, status, and structural queries.

## Reference

- **RED Method (Weaveworks, Tom Wilkie)** — the original blog post defining Rate/Errors/Duration as the standard request-service health framework; useful to cite by name in interviews.
- **OpenTelemetry Collector `spanmetrics` connector docs** (github.com/open-telemetry/opentelemetry-collector-contrib) — the reference config for deriving RED metrics from spans outside of Tempo specifically.

## Practice

- **Today's lab** (Tempo + Loki + Grafana via Docker Compose) — the most direct way to internalize trace-to-logs correlation.
- **Grafana's "Tempo and Loki" interactive demo/Play environment** (play.grafana.org) — a hosted sandbox showing a fully wired-up traces/logs/metrics correlation setup, useful for seeing what a mature production dashboard layout looks like before building your own.
- **OpenTelemetry Demo app** (github.com/open-telemetry/opentelemetry-demo) — same demo referenced on Day 94, also ships a Grafana + Tempo profile so you can practice the exact N+1/RED diagnosis workflow against a realistic multi-service app with an intentionally injected slow-query scenario in its feature flags.
