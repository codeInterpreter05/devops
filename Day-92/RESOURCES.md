# Day 92 — Resources: Loki & Log Aggregation

## Primary (assigned)

- **Grafana Loki documentation** (grafana.com/docs/loki/latest) — the assigned free starting point; start with "Loki architecture" and "LogQL" sections specifically.

## Deepen your understanding

- **"Loki: like Prometheus, but for logs" — Grafana Labs blog** (the original Loki announcement post) — explains the label-indexing design decision straight from its creators, including the explicit tradeoffs versus full-text indexing systems.
- **Fluent Bit official documentation** (docs.fluentbit.io) — the "Pipeline" concepts page (Input/Parser/Filter/Output) is the clearest explanation of its processing model.
- **Vector Remap Language (VRL) reference** (vector.dev/docs/reference/vrl) — worth skimming even if you don't deploy Vector, to see what a genuinely expressive log-transform language looks like compared to plugin-chain configs.
- **OpenTelemetry "Correlating logs, metrics, and traces" docs** (opentelemetry.io/docs/specs/otel/logs/data-model) — the standards-based explanation of how trace_id/span_id propagate into logs, which underpins the Grafana correlation workflow in this day's notes.

## Reference / lookup

- **LogQL query language reference** (grafana.com/docs/loki/latest/query) — the authoritative syntax reference for every filter/parser/metric function covered today.
- **Promtail configuration reference** (grafana.com/docs/loki/latest/send-data/promtail/configuration) — full config reference, useful even though Promtail is in maintenance mode, since it's still widely deployed.

## Practice

- **Grafana Loki's own "Getting Started" Docker Compose example** (github.com/grafana/loki/tree/main/production) — the fastest way to see Loki + Promtail + Grafana wired together without a Kubernetes cluster, useful for quick LogQL practice.
- **killercoda.com Grafana/Loki scenarios** — free browser-based sandboxes for practicing LogQL and log-shipper configuration without local setup.
