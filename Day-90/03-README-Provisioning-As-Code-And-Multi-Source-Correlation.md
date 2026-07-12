# Day 90 — Grafana Dashboards: Provisioning as Code & Loki/Tempo Correlation

**Phase:** 3 – Observability | **Week:** W15 | **Domain:** Metrics | **Flag:** ⚡ Interview-critical

## Brief

A dashboard built by clicking around in the Grafana UI lives only in that Grafana instance's database — it's not version-controlled, not reviewable in a PR, and disappears (or has to be manually recreated) if that Grafana instance is ever rebuilt. "Dashboards as code" fixes this the same way IaC fixed infrastructure: dashboards become JSON/Jsonnet files in git, deployed the same way as everything else. The second half of this file — correlating metrics with logs and traces — is what actually shortens incident response time in practice: jumping from "latency spiked" (a metric) straight to "here are the exact error logs and the exact slow trace from that window" (Loki + Tempo) without manually copy-pasting timestamps between three different tools.

## Provisioning dashboards as code

Grafana supports **provisioning**: pointing it at a directory of dashboard JSON files (and data source YAML, and alert rule YAML) that it loads automatically on startup and keeps in sync, instead of requiring manual UI clicks or API calls per environment.

```yaml
# provisioning/dashboards/dashboards.yaml
apiVersion: 1
providers:
  - name: default
    orgId: 1
    folder: 'SRE Dashboards'
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    options:
      path: /var/lib/grafana/dashboards
```

Every dashboard is just JSON — you can export any UI-built dashboard via **Dashboard settings → JSON Model**, commit that file to git, and place it in the provisioned path. In Kubernetes, this is typically done via a `ConfigMap` (or a Grafana Operator `GrafanaDashboard` CRD) mounted into the Grafana pod, so `kubectl apply -f dashboard-configmap.yaml` becomes your deployment mechanism — dashboards ship through the same CI/CD pipeline as everything else, with PR review and git history for every change.

**Why hand-writing raw dashboard JSON is painful, and what `grafonnet` fixes:** a real dashboard's JSON is often thousands of lines, with huge amounts of repeated boilerplate (every panel repeats grid position, axis config, threshold config, etc.). **Grafonnet** is a Jsonnet library that provides a much terser, composable DSL for generating that JSON — you write something like:

```jsonnet
local g = import 'g.libsonnet';
g.panel.timeSeries.new('Request Rate')
  + g.panel.timeSeries.queryOptions.withTargets([
      g.query.prometheus.new('$datasource', 'sum(rate(http_requests_total[5m])) by (status)')
    ])
```

...and Jsonnet compiles it down to the full dashboard JSON Grafana actually consumes. This matters because it lets you build **reusable panel libraries** (a "standard RED-method row" defined once, imported into every service's dashboard) instead of copy-pasting and hand-editing JSON across dozens of near-identical dashboards — exactly the same "component library" idea as shared Terraform modules or Helm subcharts.

`grafana-dashboard-exporter`-style tooling (or simply the Grafana HTTP API's `GET /api/dashboards/uid/:uid`) is how you go the other direction — pull an existing UI-edited dashboard's current JSON back out programmatically, useful for a one-time migration into version control, or for a CI check that diffs "what's live" against "what's in git" to catch dashboards that were edited directly in the UI and never committed (config drift, same failure mode as infrastructure).

## Correlating Grafana with Loki + Tempo

Grafana's real power over "just a chart renderer" comes from **cross-data-source correlation** — jumping between metrics, logs, and traces for the *same* incident window without leaving the screen.

- **Metrics → Logs**: a Loki data source can be configured with a **derived field**, or you can simply split-screen a Prometheus panel and a Loki "Logs" panel scoped to the same time range and `namespace`/`pod` variables — click-drag a time range on the metrics graph, and (with linked time ranges) the Logs panel narrows to exactly that window. In Grafana Explore, switching between a Prometheus and Loki data source preserves the selected time range so you can pivot directly from "the error-rate graph spiked at 14:32" to "here are the raw log lines from 14:30–14:35."
- **Logs → Traces**: if your logs are structured (JSON) and include a `trace_id` field (propagated from your tracing system — typically via OpenTelemetry context propagation), Loki can be configured with a **derived field** that turns any log line's `trace_id` value into a clickable link straight into the corresponding Tempo trace:

```yaml
# Loki data source provisioning, jsonData.derivedFields
jsonData:
  derivedFields:
    - datasourceUid: tempo-uid
      matcherRegex: 'trace_id=(\w+)'
      name: TraceID
      url: '$${__value.raw}'
```

- **Traces → Logs/Metrics**: Tempo, in turn, can be configured (via its own data source settings in Grafana) to link back from a trace span to the exact log lines emitted during that span (using the same `trace_id`) and to a "service graph" derived from span data showing request rate/error rate/duration per service — effectively deriving RED-method metrics straight from trace data with no separate instrumentation required.

This three-way linkage (**metrics ⇄ logs ⇄ traces**, often summarized as the "three pillars of observability" working together rather than in isolation) is precisely what shortens mean-time-to-diagnosis: instead of manually eyeballing timestamps across three separate tools and hoping they line up, one click carries the exact trace/time context from whichever pillar you started in.

## Points to Remember

- Provisioning (`provisioning/dashboards/*.yaml` + dashboard JSON files) turns dashboards into version-controlled, code-reviewed, CI/CD-deployed artifacts instead of UI-only clicks that vanish if the Grafana instance is rebuilt.
- Grafonnet is a Jsonnet DSL that compiles to dashboard JSON — its value is eliminating panel-JSON boilerplate and enabling shared, reusable panel/row libraries across dashboards.
- `disableDeletion: false` + `updateIntervalSeconds` in a provisioning config controls whether Grafana will sync away a dashboard that was manually deleted/edited outside of git — understand this setting before assuming "provisioned" means "immutable."
- Loki `derivedFields` (regex-matching a field like `trace_id` in log lines) is the mechanism that turns a raw log line into a clickable jump straight into the matching Tempo trace.
- The value of metrics/logs/traces correlation isn't the individual tools — it's collapsing the manual "cross-reference timestamps across three browser tabs" step during an incident into one click.

## Common Mistakes

- Building and tuning dashboards entirely through the UI in a "production" Grafana with no provisioning — the next Grafana redeploy (upgrade, cluster migration, disaster recovery) silently loses every manually-created dashboard.
- Provisioning dashboards but still allowing (and not noticing) manual UI edits on top of them — provisioned dashboards can usually still be edited in the UI, and without a drift-detection step in CI, "what's live" quietly diverges from "what's in git."
- Forgetting to include `trace_id` (or any correlation ID) in structured log output at all — without it, there's no mechanism (`derivedFields` or otherwise) that can link a log line back to its trace, no matter how well Grafana/Tempo/Loki are configured.
- Writing a fragile `matcherRegex` for a derived field that doesn't match the actual log line format after a logging library upgrade or format change — the trace-link silently stops working and nobody notices until they need it mid-incident.
- Treating grafonnet/dashboards-as-code as "too much tooling for now" and hand-writing JSON, then hitting the wall of unmaintainable dashboard files once the org has more than a handful of services each wanting a similar dashboard shape.
