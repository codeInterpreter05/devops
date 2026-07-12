# Day 90 — Grafana Dashboards: Data Sources & Dashboard Design Principles

**Phase:** 3 – Observability | **Week:** W15 | **Domain:** Metrics | **Flag:** ⚡ Interview-critical

## Brief

Grafana is the visualization layer that sits on top of everything you built on Day 89 (Prometheus/Alertmanager) and everything coming on Days 92–93 (Loki, and log/trace backends) — it's the single pane of glass an SRE actually stares at during an incident. Knowing Prometheus well but building unreadable, unstructured dashboards is a real and common gap: interviewers specifically probe "what makes a good SRE dashboard" because a beautifully instrumented system is worthless if the dashboard built on top of it can't answer "is this healthy right now" in under 10 seconds during a 3am page.

This day is split into three focused files:

1. **This file** — data sources and dashboard design principles (including the RED method).
2. **[02-README-Variables-Templating-And-Alerts.md](02-README-Variables-Templating-And-Alerts.md)** — reusable dashboards via variables/templating, plus Grafana's native alerting.
3. **[03-README-Provisioning-As-Code-And-Multi-Source-Correlation.md](03-README-Provisioning-As-Code-And-Multi-Source-Correlation.md)** — dashboards-as-code and correlating metrics/logs/traces (Loki + Tempo).

## Data sources

Grafana itself stores no metric/log/trace data — it's a query and visualization frontend over pluggable **data sources**: Prometheus, Loki, Tempo, Elasticsearch/OpenSearch, InfluxDB, CloudWatch, PostgreSQL/MySQL (yes, you can build dashboards straight from SQL), and dozens more via plugins.

```yaml
# Example data source config (also used for provisioning — see file 3)
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://kps-kube-prometheus-stack-prometheus:9090
    isDefault: true
    jsonData:
      timeInterval: 15s
      httpMethod: POST
```

Two settings worth understanding, not just copying:
- **`access: proxy`** (vs. `direct`) — with `proxy` (the default and almost always correct choice), the **Grafana server** makes the HTTP request to the data source, and the browser only ever talks to Grafana. With `direct`, the user's browser calls the data source URL directly — this requires the data source to be reachable from the browser (not just the Grafana pod) and to have CORS configured, which is rarely what you want in a Kubernetes cluster where Prometheus has no public endpoint.
- **`httpMethod: POST`** vs `GET` — Prometheus queries can get long (nested aggregations, many label matchers); `GET` requests have URL length limits that a complex PromQL query can exceed, silently truncating or failing. Defaulting to `POST` avoids this class of bug entirely.

A single dashboard can mix panels from multiple data sources — e.g., a "Checkout Service" dashboard with a Prometheus panel for request rate, a Loki panel for recent error logs, and a Tempo panel for a slow-trace waterfall, all on one screen (this cross-source correlation is covered in file 3).

## Dashboard design principles

The instinct when you first get Prometheus working is to add a panel for every metric you can think of. This produces a "wall of graphs" dashboard that's comprehensive and nearly useless — nobody can look at 40 panels during an incident and immediately know what's wrong. Good dashboard design is an editorial exercise, not a data-dumping exercise.

**The RED method** (for request-driven services): every service dashboard should answer three questions, in this order, at the top of the screen:
- **Rate** — how many requests/sec is this service handling right now?
- **Errors** — what fraction of those requests are failing?
- **Duration** — how long are requests taking (typically p50/p95/p99)?

```promql
# Rate
sum(rate(http_requests_total{job="checkout"}[5m]))
# Errors
sum(rate(http_requests_total{job="checkout", status=~"5.."}[5m])) / sum(rate(http_requests_total{job="checkout"}[5m]))
# Duration (p99)
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{job="checkout"}[5m])) by (le))
```

**The USE method** (for resources — nodes, disks, queues) is RED's counterpart for infrastructure rather than request-driven services:
- **Utilization** — % of time the resource is busy (CPU utilization, disk busy %).
- **Saturation** — how much extra work is queued that the resource can't service yet (run queue length, request queue depth).
- **Errors** — count of error events (disk I/O errors, dropped packets).

Why two methods, not one: RED describes the *outside view* of a service (what a client experiences), USE describes the *inside view* of a resource (what's actually straining). A checkout API can look fine on RED (low error rate, fine latency) while its underlying node is at 95% CPU saturation and about to tip over — USE-method infrastructure dashboards are what catch that *before* it becomes a RED-method incident.

**Layout and editorial principles that separate a good dashboard from a wall of graphs:**
- **Top-to-bottom = most-important-to-least-important.** The RED (or USE) summary row goes at the very top, always visible without scrolling.
- **One dashboard, one clear audience/question.** A "Checkout Service Overview" dashboard for on-call triage is a different dashboard from a "Checkout Business Metrics" dashboard for product — don't merge them.
- **Consistent units and axes.** All latency panels in milliseconds or all in seconds — not mixed — and comparable panels should share the same Y-axis scale so eyes don't get tricked by auto-scaling.
- **Annotate deploys.** A vertical marker line at deploy times (Grafana supports this natively via annotations, often pushed by CI/CD) turns "latency went up at some point" into "latency went up immediately after the 2:14pm deploy" — one of the highest-value, most underused dashboard features.
- **Red is reserved for "bad."** Don't use conditional red/green coloring cosmetically — reserve it for genuinely being able to tell health at a glance; overuse trains people to ignore color.

## Points to Remember

- Grafana stores no data itself — it's a query/render layer over data sources; `access: proxy` (server-side fetch) is the default and correct choice for almost every in-cluster setup.
- Use `httpMethod: POST` for Prometheus data sources to avoid URL-length failures on complex queries.
- RED method (Rate/Errors/Duration) for request-driven services; USE method (Utilization/Saturation/Errors) for resources/infrastructure — they answer different questions and a good on-call setup has both.
- Dashboard layout should mirror triage priority: most critical/summary info at the top, without scrolling.
- Deploy annotations are one of the highest-value, most underused dashboard features for correlating "when did this start" with "what changed."

## Common Mistakes

- Building a single "everything" dashboard with 40+ panels instead of a focused RED/USE summary — during an actual incident, nobody has time to hunt through it.
- Using `access: direct` for a data source that isn't actually reachable from end-user browsers (common in Kubernetes clusters where Prometheus has no external route), resulting in dashboards that work for the dashboard author's browser session but fail for everyone else.
- Leaving Prometheus data sources on `GET`, then getting mysterious partial/failed panel loads once a query grows past the URL length limit — usually discovered only after the dashboard has been "working fine" for months with simpler queries.
- Applying RED-method thinking to a resource (like a node) or USE-method thinking to a request-driven API — the two methods answer different questions and mixing them up produces dashboards that miss the failure mode they were supposed to catch.
- Not annotating deploys, then spending the first 20 minutes of an incident manually cross-referencing a deploy log against a graph's timestamp axis.
