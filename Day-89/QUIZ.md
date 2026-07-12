# Day 89 — Quiz: Prometheus Fundamentals

Try to answer without looking at your notes. Answers are at the bottom.

1. What uniquely identifies a Prometheus time series?
2. Why should you (almost) never query a raw counter value directly? What do you wrap it in instead?
3. Why can't you average `quantile` values from a summary metric across multiple pod replicas, but you can sum histogram bucket counts across replicas?
4. What is the minimum recommended relationship between a `rate()` window and the scrape interval, and why?
5. Why must the `le` label survive a `sum(...) by (...)` aggregation before calling `histogram_quantile()`?
6. What's the difference between `relabel_configs` and `metric_relabel_configs`, and when would you use each?
7. What happens to `__meta_kubernetes_*` labels if you don't explicitly promote them via `relabel_configs`?
8. In Alertmanager routing, what does `continue: true` do, and give a scenario where you'd need it?
9. What's the difference between an Alertmanager silence and an inhibition rule?
10. Why does an alerting rule need a `for:` duration, and what happens without one?
11. What are recording rules for, and what naming convention does the community use for them?
12. **Interview question:** What is the difference between a counter and a gauge? When would you use a histogram vs summary?

---

## Answers

1. The combination of the **metric name** plus its full **label set**. Changing even one label's value produces a distinct, independently-stored time series — they just happen to share a name for query/aggregation purposes.
2. A raw counter only ever increases (except on process restart), so its absolute value ("27,183 requests since start") isn't actionable on its own. You wrap it in `rate()` (per-second average rate, handles counter resets) or `increase()` (total increase over a window) to get something meaningful to graph or alert on.
3. Quantiles are not mathematically composable — you cannot average five different p99 estimates and get a valid combined p99. Histogram bucket *counts*, however, are just counts of observations falling in a range, and counts sum cleanly across instances; `histogram_quantile()` can then be run once on the combined cumulative buckets to produce a mathematically valid cluster-wide percentile.
4. At least **4x the scrape interval** — e.g., with a 15s scrape interval, use `[1m]` at minimum, `[5m]` is more common. Shorter windows contain too few samples, making `rate()` noisy or occasionally return "no data" if a scrape is missed.
5. `histogram_quantile()` interpolates a percentile by walking the cumulative bucket boundaries (`le` = "less than or equal to") to find where the target percentile's count falls. If `le` is dropped during aggregation, the buckets collapse into a single value with no boundary information left to interpolate across, and the function can't produce a meaningful answer.
6. `relabel_configs` run **before scraping**, against target metadata — they decide whether to scrape a target at all and how to construct its address/path. `metric_relabel_configs` run **after scraping**, against the returned samples — they're used to drop/rewrite specific metrics you've already paid the network cost to fetch. Use `relabel_configs` to filter out entire targets early (cheaper); use `metric_relabel_configs` when you need to inspect actual metric names/values to decide what to drop.
7. They're **dropped by default** before the sample is stored — they exist only transiently during the discovery/relabeling phase. If you want a piece of Kubernetes metadata (namespace, pod name, node, etc.) available on your stored metrics, you must explicitly copy it into a real label via a `relabel_configs` rule (`target_label:`).
8. `continue: true` lets route evaluation keep going into sibling/child routes even after a route already matched, instead of stopping at the first match. Example: a critical alert needs to go to PagerDuty *and* to a team's Slack channel — without `continue: true` on the PagerDuty route, the Slack route would never be evaluated.
9. A **silence** is a manually created, time-bounded mute a human sets up (e.g., during planned maintenance) and it expires on its own. An **inhibition rule** is a standing, always-on config rule that automatically suppresses one class of alert whenever another (matching, related) alert is already firing — no human action needed each time.
10. `for:` requires the alert condition to remain true continuously across that entire duration before transitioning from `pending` to `firing`. Without it, a single noisy evaluation cycle (a brief blip) would immediately page someone even though the issue self-resolved before anyone could act — pure alert noise.
11. Recording rules precompute and store the result of a (usually expensive or frequently reused) PromQL expression as its own time series, evaluated on a schedule. This improves dashboard/alert query performance and guarantees dashboards and alerts referencing the same logic never silently drift apart. Convention: `level:metric:operation`, e.g. `job:http_requests:error_ratio5m`.
12. Strong answer: "A counter only ever increases (or resets to zero on restart) — it's for counting things like total requests or total errors, and you always query it through `rate()` or `increase()` rather than its raw value. A gauge can go up or down and represents a current value like memory usage or queue depth — you query it directly. For distributions like latency, I'd default to a histogram over a summary because histogram buckets can be aggregated across replicas with `sum()` and then run through `histogram_quantile()` to get a valid cluster-wide percentile — summary quantiles are computed client-side and can't be mathematically combined across instances, which matters a lot once you have more than one replica of a service, which is basically always in Kubernetes."
