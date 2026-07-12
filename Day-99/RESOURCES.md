# Day 99 — Resources: AWS-native Observability

## Primary (assigned)

- **AWS Observability documentation** (aws-observability.github.io / docs.aws.amazon.com) — the assigned starting point for this day; covers Container Insights, X-Ray, ADOT, Synthetics, and Amazon Managed Prometheus/Grafana from AWS's own reference architecture perspective.

## Deepen your understanding

- **AWS Distro for OpenTelemetry docs** (aws-otel.github.io) — the ADOT-specific documentation, including collector configuration examples and language-specific auto-instrumentation guides. Worth reading before your first real ADOT deployment.
- **"CloudWatch Logs Insights query syntax" AWS docs page** — the authoritative reference for every `fields`/`filter`/`stats`/`parse` operator; bookmark this, you'll come back to it constantly when writing non-trivial queries.
- **OpenTelemetry.io docs** — not AWS-specific, but essential for understanding the vendor-neutral concepts (spans, resources, semantic conventions) that ADOT is built on top of.
- **AWS Prescriptive Guidance: "Amazon Managed Service for Prometheus and Amazon Managed Grafana"** — a good concise treatment of the managed-vs-self-managed trade-off from AWS's own architects, useful cross-reference against the framing in this repo's notes.

## Reference / lookup

- `aws eks create-addon`, `aws logs start-query`/`get-query-results`, `aws xray get-service-graph`, `aws amp create-workspace`, `aws grafana create-workspace` — CLI reference pages for each service used today.
- **X-Ray sampling rules reference** — the exact schema and precedence rules for writing custom sampling rules per service/route.

## Practice

- Take a workload you already have running on EKS (from an earlier day's lab) and instrument it end-to-end: Container Insights for metrics, a Logs Insights saved query for errors, ADOT for tracing into X-Ray, and one Synthetics canary hitting its main endpoint. Doing all four against the same real workload makes the trade-offs in this day's notes concrete instead of abstract.
