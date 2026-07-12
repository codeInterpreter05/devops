# Day 90 — Quiz: Grafana Dashboards

Try to answer without looking at your notes. Answers are at the bottom.

1. Does Grafana store metric/log/trace data itself? What is its actual role in the observability stack?
2. What's the difference between `access: proxy` and `access: direct` for a data source, and which should you default to in Kubernetes?
3. Why should Prometheus data sources typically use `httpMethod: POST` rather than `GET`?
4. What are the three questions the RED method answers, and for what kind of service is it meant?
5. What does the USE method measure, and why is it a *different* set of questions than RED?
6. What PromQL function populates a dashboard variable's dropdown dynamically, and what happens to newly-created resources (e.g., a new namespace) if the variable's refresh is set to "Never"?
7. Why must a multi-value variable's query use `=~` instead of `=`?
8. What is "dashboards as code" / provisioning, and what real failure mode does it prevent?
9. What problem does grafonnet solve that hand-written dashboard JSON doesn't?
10. What is a Loki `derivedField`, and what does it let you click through to?
11. When would you choose Grafana-managed alerting over a Prometheus `PrometheusRule`, and what's the recommended pattern for where the actual notification routing should live?
12. **Interview question:** What makes a good SRE dashboard? What is the RED method and how do you apply it in Grafana?

---

## Answers

1. No — Grafana stores no metric/log/trace data of its own. It's a query and visualization frontend over pluggable data sources (Prometheus, Loki, Tempo, CloudWatch, SQL databases, etc.); all actual storage and query execution happens in those backends.
2. `access: proxy` means the Grafana **server** makes the request to the data source, and the user's browser only ever talks to Grafana — this is the default and correct choice almost everywhere, especially in Kubernetes where the data source (e.g., Prometheus) has no route reachable from end-user browsers. `access: direct` means the browser calls the data source URL itself, requiring it to be externally reachable and CORS-configured.
3. Complex PromQL queries (nested aggregations, many label matchers) can exceed URL length limits under `GET`, causing silent truncation or failures; `POST` sends the query in the request body with no such limit.
4. Rate (requests/sec), Errors (fraction of requests failing), Duration (latency, typically p50/p95/p99) — meant for request-driven services (APIs, web services) where a client experiences discrete requests.
5. USE measures Utilization (% busy), Saturation (queued work the resource can't yet service), and Errors — it's meant for resources/infrastructure (CPU, disk, network, queues), answering "is this resource straining," which is a different question than "is the service serving requests well" (RED). A resource can be saturating under USE while the service still looks fine under RED, right before it tips into a RED-visible incident.
6. `label_values(metric, label)` — it returns the distinct values a label has taken across all matching series. If refresh is set to "Never," the variable's query only runs once (e.g., at dashboard creation/edit time) and newly created resources never appear in the dropdown until refresh is changed to "On Dashboard Load" or "On Time Range Change."
7. Grafana converts multiple selected values into a regex alternation (e.g., `prod-a|prod-b`) for multi-value variables. `=` performs an exact string match, which only works for a single literal value; `=~` performs a regex match, which is required to correctly OR multiple selected values together.
8. Provisioning means Grafana loads dashboard JSON (and data source/alert configs) automatically from files (often backed by a git-tracked ConfigMap in Kubernetes) rather than requiring manual UI creation. It prevents dashboards from being lost when a Grafana instance is rebuilt/redeployed, and makes dashboard changes reviewable via PRs with full git history, the same way IaC prevents infrastructure drift and loss.
9. Hand-written dashboard JSON is thousands of lines with heavy repeated boilerplate per panel (grid position, axis config, thresholds, etc.). Grafonnet is a Jsonnet DSL that compiles down to that JSON but lets you define reusable, composable panel/row functions once and reuse them across many dashboards — avoiding copy-paste drift across near-identical dashboards.
10. A derived field matches a regex (e.g., `trace_id=(\w+)`) against log line content in a Loki data source and turns the matched value into a clickable link — typically configured to jump directly into the matching trace in a Tempo data source, correlating a log line to its full distributed trace with one click.
11. Choose Grafana-managed alerting when you need to alert on a data source Prometheus's own alerting rules can't query directly — Loki log patterns, CloudWatch metrics, SQL query results, etc. For anything already in Prometheus, prefer Prometheus/Alertmanager rules since that's the industry-standard pattern most on-call tooling assumes. Recommended pattern: forward Grafana-managed alert notifications to the same external Alertmanager already used by Prometheus, so there's one authoritative system for routing, grouping, inhibition, and silences instead of two independently-configured, potentially inconsistent ones.
12. Strong answer: "A good SRE dashboard answers 'is this healthy right now' in seconds, not minutes — it's an editorial choice, not a data dump of every available metric. I apply the RED method — Rate, Errors, Duration — as the top row of any request-driven service's dashboard: request rate via `sum(rate(http_requests_total[5m]))`, error ratio via `sum(rate(..., status=~"5..")[5m])) / sum(rate(...[5m]))`, and p99 latency via `histogram_quantile(0.99, sum(rate(..._bucket[5m])) by (le))`. That summary sits at the very top with no scrolling required, I use variables so the same dashboard works across every namespace/service without copy-pasting, and I annotate deploys so a latency spike can immediately be correlated with whatever changed. For infrastructure-level panels (nodes, disks), I switch to the USE method instead, since RED doesn't apply to things that aren't handling discrete requests."
