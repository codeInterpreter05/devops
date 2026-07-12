# Day 101-109 — Observability Capstone I: The Full Stack Architecture (Prometheus + Grafana + Loki + Tempo on Kubernetes)

**Phase:** 3 – Observability | **Week:** W17-W18 | **Domain:** Review | **Flag:** 📌

## Brief

This is a 9-day capstone block that closes out Phase 3. It's not new theory — it's proof you can actually stand up and operate what the last several weeks of Phase 3 taught in pieces (metrics, logs, tracing, alerting) as one integrated platform, on Kubernetes, the way it's actually run in production. In parallel, you're also studying for and sitting the AWS SAA-C03 exam, because Phase 4 assumes solid AWS fundamentals and this is the assigned structured syllabus for getting there.

This block is split into three files:

1. **This file** — the architecture of the full observability stack: how Prometheus, Grafana, Loki, and Tempo fit together on K8s, and the correlation mechanisms (exemplars, derived fields, trace-to-metrics-to-logs) that make "three separate tools" behave like one system.
2. **[02-README-SLOs-ErrorBudgets-PagerDuty-Runbooks.md](02-README-SLOs-ErrorBudgets-PagerDuty-Runbooks.md)** — defining SLOs/SLIs for a real service, error-budget math, multi-window burn-rate alerting, PagerDuty on-call escalation, and what makes a runbook actually usable at 3am.
3. **[03-README-AWS-SAA-C03-Exam-Day.md](03-README-AWS-SAA-C03-Exam-Day.md)** — final-week prep checklist, Pearson VUE exam-day logistics, and the last-mile mistakes that cost people a passing score.

The hands-on build-out (deploying the stack, instrumenting an app, writing SLO alerts, wiring PagerDuty) lives in [LAB.md](LAB.md) — read this file first for the *why* behind each component, then go build it there.

## Why three pillars, and why they don't replace each other

**Metrics** (Prometheus) answer "is something wrong, and how bad, right now, in aggregate" — cheap to store, cheap to query over long time ranges, but they tell you *that* p99 latency spiked, not *why*. **Logs** (Loki) answer "what exactly happened on this one request/instance" — high-cardinality, high-detail, but expensive to search broadly and impossible to correlate at scale without discipline. **Traces** (Tempo) answer "where did the time go across services for this one request" — they show you the causal chain (service A called service B which was slow) that neither metrics nor logs can show on their own.

A real incident workflow uses all three in sequence: a **metric** alert fires (error rate up on `checkout-service`) → you jump to a **trace** for a slow/failed request to see which downstream call is the actual bottleneck → you jump to the **logs** for that specific span/pod to see the exact error message and stack trace. The entire point of this capstone is wiring that jump so it's three clicks, not three separate tools with no connective tissue.

## The stack, component by component

```
                     ┌─────────────────────────┐
                     │        Grafana           │  <- unified frontend, one pane of glass
                     │  (dashboards + explore)   │
                     └───────────┬───────────────┘
                    ┌────────────┼────────────────┐
                    ▼             ▼                 ▼
             ┌─────────────┐ ┌─────────┐    ┌──────────────┐
             │ Prometheus  │ │  Loki    │    │    Tempo     │
             │  (metrics)  │ │  (logs)  │    │   (traces)   │
             └──────┬──────┘ └────┬─────┘    └──────┬───────┘
                    │ scrape       │ push            │ push (OTLP)
                    │ (pull)       │                 │
         ┌──────────┴───┐   ┌──────┴───────┐  ┌──────┴────────┐
         │ ServiceMonitor│  │  Promtail /   │  │ OTel Collector │
         │ / PodMonitor  │  │ Grafana Agent │  │  (receives &   │
         │  (via Operator)│  │ (log shipper) │  │  fans out)     │
         └──────────┬────┘   └──────┬───────┘  └──────┬────────┘
                    │                │                  │
                    └────────────────┴──────────────────┘
                                     │
                          ┌──────────▼───────────┐
                          │   Instrumented app     │
                          │ (OpenTelemetry SDK:     │
                          │  metrics + logs + traces)│
                          └─────────────────────────┘
```

- **Prometheus** — pull-based metrics. It scrapes `/metrics` HTTP endpoints on a schedule. On Kubernetes, you almost never hand-configure scrape targets; the **Prometheus Operator** (bundled in the `kube-prometheus-stack` Helm chart) watches `ServiceMonitor` and `PodMonitor` custom resources and auto-generates scrape config from them. This is the single biggest quality-of-life difference between "raw Prometheus" and "Prometheus on Kubernetes."
- **Grafana** — the unified frontend. It doesn't store any data itself; it queries Prometheus, Loki, and Tempo as separate **data sources** and renders them in one UI. Dashboards can be provisioned as code (ConfigMaps with a `grafana_dashboard: "1"` label, picked up by a sidecar container) instead of clicked together by hand — treat dashboards like Kubernetes manifests: version-controlled, reviewed, applied via CI.
- **Loki** — push-based log aggregation, deliberately designed **not** to index log line content. It only indexes a small set of **labels** (e.g., `namespace`, `pod`, `container`, `app`) — the actual log text is compressed and stored cheaply, searched with `grep`-like filters at query time. This is the opposite indexing philosophy from Elasticsearch, and it's why Loki is dramatically cheaper to run at scale: **never** turn a high-cardinality field (request ID, user ID, trace ID) into a Loki *label* — that recreates the cardinality explosion Loki was built to avoid. High-cardinality identifiers belong in the log *line content* (searchable via `|=` / `|~` filters or extracted with `| json`/`| logfmt`), not the label set.
- **Promtail / Grafana Agent** — the log shipper. Runs as a DaemonSet, tails container log files from the node (`/var/log/pods/...`), attaches Kubernetes metadata as labels (namespace, pod, container), and pushes batches to Loki.
- **Tempo** — trace storage and query backend. Receives spans over OTLP (OpenTelemetry Protocol), stores them cheaply in object storage (S3/GCS/local disk), and is deliberately "dumb" about indexing beyond trace ID — search is delegated to metrics-generated indexes or exemplars rather than Tempo doing expensive full-text search over span attributes itself.
- **OpenTelemetry Collector** — the ingestion and fan-out layer. Applications emit metrics/logs/traces over OTLP to the Collector; the Collector's pipeline config decides where each signal goes (metrics → Prometheus via remote-write or a scrape-able endpoint, logs → Loki, traces → Tempo). This decouples your application code from your backend choice — swap Tempo for another trace backend later and the app doesn't change, only the Collector's exporter config does.
- **The instrumented app** — uses an OpenTelemetry SDK (auto-instrumentation agents exist for Java, Python, Node, .NET, Go) to emit all three signal types with **shared context** (the same trace ID and span ID available to the metrics exemplar and the log line) — this shared context is what makes correlation possible at all.

## Correlation: making three tools behave like one

This is the actual engineering payoff of this capstone, and the part most tutorials skip:

- **Exemplars (metrics → traces).** A Prometheus histogram (e.g., `http_request_duration_seconds_bucket`) can attach an **exemplar** to a sample — a trace ID for one specific request that landed in that latency bucket. In Grafana, when you view a latency graph with exemplars enabled, you see small dots on the graph; clicking one jumps straight to that exact trace in Tempo. This turns "p99 spiked at 14:32" into "here is the actual slow request" in one click, instead of guessing which trace to search for.
- **Derived fields (logs → traces).** Configure Loki as a Grafana data source with a **derived field**: a regex that extracts a `trace_id` from the log line, rendered as a clickable link that opens that trace in Tempo. Requires your app to actually log the trace ID on each line (standard structured-logging practice with OpenTelemetry: inject trace context into the logger).
- **Trace to logs / trace to metrics (traces → logs/metrics).** Configure Tempo's data source in Grafana with "trace to logs" pointing at Loki (filtered by the span's `service.name` and time range) and "trace to metrics" pointing at Prometheus — so from an open trace you can jump to the logs emitted during that exact span, or the service's aggregate metrics for that time window.
- **Shared labels as the glue.** None of this works if your metrics, logs, and traces don't agree on what to call a service. Standardize on OpenTelemetry semantic conventions (`service.name`, `service.namespace`, `deployment.environment`) everywhere — Prometheus job/instance labels, Loki labels, and Tempo resource attributes should all resolve to the same service identity, or your correlation links silently return empty results.

## Points to Remember

- Metrics tell you *that* and *how much*; logs tell you *what exactly*; traces tell you *where the time went across services* — you need all three, and the value is in wiring them together, not just running three tools side by side.
- Prometheus is pull/scrape-based and driven by `ServiceMonitor`/`PodMonitor` CRDs on Kubernetes; Loki and Tempo are push-based, receiving data shipped to them by an agent/collector.
- Loki indexes **labels only**, never log content — keep labels low-cardinality (namespace/pod/app), and never promote a request ID or trace ID to a label.
- The OpenTelemetry Collector is the fan-out point that decouples app instrumentation from backend choice — apps talk OTLP to the Collector, the Collector's exporters talk to Prometheus/Loki/Tempo.
- Exemplars, derived fields, and trace-to-logs/metrics links are what make this "one observability platform" instead of "three dashboards you tab between" — and they all depend on a shared identity (trace ID, `service.name`) propagated consistently through every signal.

## Common Mistakes

- Adding high-cardinality data (user ID, request ID, trace ID) as a Loki **label** instead of leaving it in the log body — this silently recreates the exact cost/cardinality problem Loki's design avoids, and can make Loki itself fall over under load.
- Standing up Prometheus, Loki, and Tempo but never configuring exemplars/derived fields/trace-to-logs links — technically "the stack is deployed," but nobody gets the actual cross-signal correlation that's the point of running all three.
- Letting each team's service use a different label/field name for the same concept (`service`, `app`, `service_name`, `svc`) across metrics/logs/traces — correlation queries and dashboards silently break or return partial data across teams.
- Treating Tempo like a full-text search engine over span attributes — it's optimized for trace-ID lookup and metrics-generated indexes (TraceQL helps, but it's not a substitute for good exemplars and derived-field linking from metrics/logs).
- Forgetting that Grafana stores no data itself — if Grafana "loses a dashboard," the actual data in Prometheus/Loki/Tempo is untouched; the fix is re-provisioning the dashboard definition, not panicking about data loss.
