# Day 90 — Resources: Grafana Dashboards

## Primary (assigned)

- **Grafana documentation — Dashboard best practices** (grafana.com/docs/grafana/latest/best-practices/best-practices-for-creating-dashboards) — the assigned free starting point; covers layout, variable usage, and design guidance directly from the maintainers.

## Deepen your understanding

- **"The RED Method: How to Instrument Your Services" — Weaveworks blog** (the original write-up by Tom Wilkie that coined the RED method) — short, foundational, explains the reasoning behind Rate/Errors/Duration.
- **Brendan Gregg's USE Method page** (brendangregg.com/usemethod.html) — the original source for Utilization/Saturation/Errors, written by the performance engineer who coined it; includes worked examples per resource type (CPU, disk, network).
- **Grafonnet documentation** (grafana.github.io/grafonnet) — official docs for the Jsonnet dashboard-generation library, with worked examples of reusable panel functions.
- **Grafana provisioning docs** (grafana.com/docs/grafana/latest/administration/provisioning) — the authoritative reference for dashboard, data source, and alerting provisioning file formats.

## Reference / lookup

- **Grafana HTTP API docs** (grafana.com/docs/grafana/latest/developers/http_api) — for exporting/importing dashboard JSON programmatically, useful for CI drift checks.
- **Grafana Labs' "correlations" and Loki `derivedFields` docs** — reference for wiring metrics/logs/traces together.

## Practice

- **play.grafana.org** — Grafana's own free public demo instance with real dashboards (including RED/USE-style examples) you can inspect and reverse-engineer without deploying anything.
- **killercoda.com / Grafana Labs' own katacoda-style scenarios** — free browser-based sandboxes for practicing dashboard building against live Prometheus/Loki data without local setup.
