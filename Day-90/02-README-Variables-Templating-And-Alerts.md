# Day 90 — Grafana Dashboards: Variables, Templating & Alerts

**Phase:** 3 – Observability | **Week:** W15 | **Domain:** Metrics | **Flag:** ⚡ Interview-critical

## Brief

Without variables, you'd need a separate hand-built dashboard for every cluster, every namespace, every service — an unmaintainable multiplication. Templating is what turns "one dashboard per team, forever out of date" into "one dashboard, reusable across your entire fleet, that stays in sync automatically as new services/pods appear." Grafana's built-in alerting (distinct from Alertmanager) is the newer piece — knowing when to use each is a question that comes up because teams often run both without understanding why, doubling their alerting surface unintentionally.

## Variables and templating

A **dashboard variable** is a placeholder, populated from a query (or a static list), that every panel's query can reference — changing the variable at the top of the dashboard re-renders every panel without editing a single query.

```
Variable name: namespace
Type: Query
Data source: Prometheus
Query: label_values(kube_pod_info, namespace)
Refresh: On Dashboard Load
```

Then any panel's query references it with `$namespace` (or `[[namespace]]` in older syntax):

```promql
sum(rate(http_requests_total{namespace="$namespace"}[5m])) by (pod)
```

- **`label_values(metric, label)`** is the workhorse query function for populating a variable — it returns the distinct values a label has taken across all series of that metric, so as new namespaces/pods/services appear, they automatically show up in the dropdown with zero dashboard edits.
- **Multi-value variables** let a user select several values at once (e.g., "namespace: prod-a, prod-b"). PromQL then needs the **regex** form Grafana auto-generates: `namespace=~"$namespace"` (note `=~`, not `=`) so multiple selected values get OR'd together as a regex alternation (`prod-a|prod-b`) instead of a literal string match, which would only ever match if exactly one value happened to equal the whole comma-joined string.
- **Chained variables**: a `pod` variable's query can depend on the already-selected `namespace` variable — `label_values(kube_pod_info{namespace="$namespace"}, pod)` — so the pod dropdown only shows pods that actually exist in the currently-selected namespace, rather than every pod across the whole cluster.
- **`$__interval`** — a built-in variable Grafana computes automatically based on the panel's time range and width, letting a `rate(x[$__interval])` query automatically use a coarser window when zoomed out over a week and a finer window when zoomed into the last hour — one query definition that adapts to whatever range the viewer picks.

## Alerts in Grafana

Grafana ships its own alerting engine (Grafana-managed alert rules), separate from Prometheus/Alertmanager alerting rules covered on Day 89. A Grafana alert rule can query **any** configured data source — not just Prometheus — which is its main advantage: you can alert directly on a Loki log-count query, a CloudWatch metric, or a SQL query result, none of which Prometheus's own alerting rules can touch since those only evaluate PromQL against Prometheus's own storage.

```
Alert rule: HighLoginFailureRate
Query (Loki): sum(count_over_time({app="auth"} |= "login failed" [5m]))
Condition: IS ABOVE 50
Evaluate every: 1m for 5m
Contact point: checkout-slack (routed via a Grafana notification policy, structurally similar to Alertmanager's routes)
```

Grafana alerting can also be configured to **forward to an external Alertmanager** (including the same one Prometheus already uses) instead of using Grafana's own built-in notification routing — this is the recommended pattern in most serious setups, because it keeps a single source of truth for routing/grouping/inhibition/silences rather than maintaining two separate, potentially inconsistent alert-routing systems.

**When to use Grafana alerting vs. Prometheus alerting rules:**
- Alerting purely on Prometheus metrics with routing needs Alertmanager already handles well (inhibition, silences, existing receivers)? Keep it in Prometheus/Alertmanager — it's closer to the data, evaluated by the same process scraping it, and is the industry-standard pattern most on-call tooling (PagerDuty integrations, runbooks) assumes.
- Need to alert on a non-Prometheus data source (Loki log pattern, CloudWatch metric, a raw SQL check)? That's exactly what Grafana-managed alert rules are for.
- Avoid duplicating the *same* alert in both systems — pick one authority per alert to prevent double-paging or silences applied in one system not affecting the other.

## Points to Remember

- `label_values(metric, label)` populates a dashboard variable dynamically — new pods/namespaces appear automatically with no dashboard edits.
- Multi-value variables require `=~` (regex match) in PromQL, not `=` — Grafana handles the conversion, but you must reference the variable with the right operator in your query.
- Chained variables (a child variable's query filtered by a parent variable's current selection) keep dropdowns relevant instead of showing irrelevant combinations.
- `$__interval` lets a single panel query adapt its window size to the viewer's selected time range automatically.
- Grafana-managed alerts can query any data source (Loki, CloudWatch, SQL, etc.), not just Prometheus — use them for exactly that case, and prefer forwarding to a shared Alertmanager over maintaining two independent alert-routing systems.

## Common Mistakes

- Using `=` instead of `=~` for a multi-value variable in a PromQL query — it silently only matches when a single value is selected, and returns empty results the moment a user selects more than one.
- Hardcoding a namespace/pod/service name into a dashboard instead of using variables — the dashboard then needs to be copy-pasted and manually edited for every new environment or team, and copies drift out of sync over time.
- Setting a variable's refresh to "Never" then wondering why newly-created namespaces/pods never appear in the dropdown — it needs "On Dashboard Load" (or "On Time Range Change") to re-run its `label_values()` query.
- Running the *same* logical alert in both Grafana-managed alerting and a Prometheus `PrometheusRule`, causing double notifications, and confusion during an incident about which system's silence actually suppressed anything.
- Building deeply chained variables (5+ levels) that make the dashboard slow to load because each level triggers its own metadata query — keep chains shallow (1–2 levels) for anything meant to load quickly during an incident.
