# Day 100 — Quiz: Datadog / New Relic (Commercial Tools)

Try to answer without looking at your notes. Answers are at the bottom.

1. What three jobs does the Datadog Agent do on each host/node it runs on?
2. Why is the Datadog Agent deployed as a Kubernetes DaemonSet rather than a regular Deployment?
3. What does the Cluster Agent do that the per-node Agent deliberately does not, and why does that separation exist?
4. What is Autodiscovery, and what's the trade-off it introduces?
5. In a flame graph, what does the *width* of a span bar represent, and how do you use that to find a bottleneck?
6. What is the Datadog Service Map built from, and what's a limitation of trusting it completely?
7. Why do anomaly monitors and composite monitors exist — what specific operational problem are they solving?
8. Explain Datadog's two-tier log model (indexed vs. archived) and why it exists.
9. What is "trace-to-log correlation" and why is it a good example of where commercial platform value actually comes from?
10. What is NRQL, and how does New Relic's architecture (NRDB) differ conceptually from Datadog's more product-segmented backends?
11. **Interview question:** You have an unlimited budget. Would you choose Datadog or a self-managed Prometheus stack? Justify.
12. Name one concrete cost trap that applies to *both* commercial platforms (Datadog/New Relic) and self-managed Prometheus alike.

---

## Answers

1. Collecting host-level metrics (CPU/memory/disk/network via built-in system checks), running integration checks (300+ plugins for specific technologies like Postgres/Redis/Nginx), and acting as a local aggregation/forwarding point for custom metrics (DogStatsD) and APM traces (via its local trace-agent component).
2. A DaemonSet guarantees exactly one Agent pod per node automatically, including new nodes added later — a regular Deployment gives you no such guarantee and could leave some nodes with zero visibility.
3. The Cluster Agent handles cluster-wide work: talking to the Kubernetes API server for cluster metadata, computing HPA custom/external metrics, and acting as a single fan-in point of contact. This separation exists so hundreds of node-level Agents don't all independently hit the Kubernetes API server, which would put unnecessary load on it at scale.
4. Autodiscovery is the mechanism where the Agent watches the Kubernetes API for pod events and matches pod annotations against known integration templates to automatically start/stop checks as pods come and go — no static per-pod config needed. The trade-off is that this convenience is tied to Datadog's proprietary annotation schema, coupling your manifests to that vendor's format.
5. Width represents time spent in that span (duration), drawn proportionally. The widest bar low in the call stack is your bottleneck — visually obvious without correlating timestamps manually, unlike reconstructing a timeline from scattered log lines.
6. It's auto-generated purely from observed trace data — no manual maintenance. The limitation: it's only as complete as your instrumentation coverage, so an uninstrumented service/hop simply doesn't appear, which can hide a genuinely broken dependency rather than flag it.
7. Static thresholds are the biggest source of alert fatigue — they either miss real problems during naturally low-traffic periods or fire constantly during expected traffic spikes/cyclical patterns. Anomaly monitors use seasonal/trend baselining instead of a fixed number; composite monitors require multiple corroborating signals (boolean combination of monitors) before alerting — both exist specifically to cut false-positive page volume.
8. Indexed logs are fully parsed/tagged/searchable in the Log Explorer and count toward billed retention — you choose what gets indexed via indexing/exclusion filters. Archived logs are cheaply stored (e.g., in your own S3 bucket) without being searchable, and can be "rehydrated" (pulled back into the indexed tier) on demand for a specific time range/query. This exists because indexing 100% of raw log volume at real traffic scale is cost-prohibitive.
9. Because traces and logs both flow through the same Agent and carry the same trace ID, you can jump directly from a slow span in a flame graph to the exact log lines emitted during that span, with no manual correlation. It's a good example of commercial value coming from *integration between pillars* (metrics/traces/logs sharing identifiers/tags) rather than any single pillar being individually better than a good OSS tool.
10. NRQL is New Relic's SQL-like query language used across all its telemetry types. Architecturally, New Relic centers on a single unified datastore (NRDB) that metrics, events, logs, and traces all land in and get queried from with one language — versus Datadog's more product-segmented backends (separate systems/query approaches per pillar, tied together via shared tags/trace IDs rather than one underlying store).
11. Strong answer: with unlimited budget, the optimization target shifts from minimizing cost to minimizing engineer-hours spent building/maintaining observability tooling and minimizing mean-time-to-resolution during incidents. Datadog wins decisively on both under that framing — native cross-pillar correlation (flame graph → log lines → related metric, one click), out-of-the-box dashboards for every integration, and zero operational burden (no HA/scaling/upgrades to own) versus assembling and integrating Prometheus + Alertmanager + Grafana + Loki/Tempo yourself. The answer should explicitly name that unlimited budget changes what's being optimized — that's the signal an interviewer is listening for, not a flat "Datadog is better" or "OSS is always better" statement.
12. High-cardinality tags/labels (e.g., tagging metrics or logs with unique request IDs, user IDs, or other high-cardinality dimensions) blow up cost in both worlds — it drives up per-series billing in Datadog/New Relic just as it drives up storage/query cost (and can crash) a self-managed Prometheus TSDB. Cardinality discipline matters regardless of which platform you're on.
