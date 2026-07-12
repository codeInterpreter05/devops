# Day 94 — Resources: OpenTelemetry

## Primary (assigned)

- **OpenTelemetry documentation** (opentelemetry.io/docs) — free, the assigned starting point for this day. Covers concepts, the Python SDK, and Collector configuration authoritatively and is kept current with the (fast-moving) spec.

## Deepen your understanding

- **OpenTelemetry Python instrumentation docs** (opentelemetry.io/docs/languages/python/instrumentation) — the exact auto/manual instrumentation APIs used in this day's lab.
- **W3C Trace Context specification** (w3.org/TR/trace-context) — the actual standard behind `traceparent`/`tracestate`; worth reading the header format section once so you can explain it from first principles, not from memory.
- **"Distributed Tracing in Practice"** (O'Reilly, Parker/Spoonhower/Mace/Brutlag) — the deepest treatment of span models, sampling tradeoffs, and Collector-style pipeline design; written by engineers who built tracing systems at scale (Twitter, Uber, Lightstep).
- **OpenTelemetry Collector Contrib repo** (github.com/open-telemetry/opentelemetry-collector-contrib) — browse the actual receiver/processor/exporter implementations to see what's available beyond the core distribution (e.g., tail_sampling processor source and its README are the clearest spec for how it actually behaves).

## Reference

- **OpenTelemetry Specification** (github.com/open-telemetry/opentelemetry-specification) — the ground-truth data model definitions for spans, metrics, and logs when docs are ambiguous.
- **Collector configuration docs** (opentelemetry.io/docs/collector/configuration) — full reference for every receiver/processor/exporter and pipeline wiring syntax.

## Practice

- **Instrument a FastAPI/Flask app end-to-end** (today's lab) — the single best way to internalize auto vs. manual instrumentation and see a real trace tree.
- **OpenTelemetry Demo app** (github.com/open-telemetry/opentelemetry-demo) — a full microservices e-commerce app, pre-instrumented with OTel across a dozen languages/services, with Jaeger/Grafana wired up via Docker Compose — excellent for seeing multi-service context propagation and tail sampling in a realistic topology without building it yourself.
