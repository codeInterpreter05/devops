# Day 92 — Quiz: Loki & Log Aggregation

Try to answer without looking at your notes. Answers are at the bottom.

1. What is the core architectural difference between how Loki and Elasticsearch/OpenSearch handle log content at ingest time?
2. Why does Loki's design deliberately mirror Prometheus's (labels, query language, operational model)?
3. What does the `|=` operator do in LogQL, and how is it different from `|~`?
4. How can you filter or aggregate on a JSON field in a log line if Loki doesn't index log content at all?
5. Write a LogQL query that returns the rate of lines containing "error" per second, over a 5-minute window, for `{app="api"}`.
6. What does `unwrap` do in LogQL, and why is it needed before functions like `quantile_over_time()`?
7. What is Promtail's current ecosystem status, and what is it being replaced by?
8. Name one concrete reason a team might choose Fluent Bit over Vector, and one reason they might choose Vector over Fluent Bit.
9. Why is adding a `trace_id` or `user_id` as an actual Loki *label* dangerous, and where should that value go instead?
10. What Loki-specific cost (beyond "too many index entries," which also applies to Prometheus) does high cardinality add on top?
11. What field must appear in structured logs for the "click a log line, jump to its trace" Grafana workflow to work at all?
12. **Interview question:** Compare Loki and Elasticsearch for log storage. What are the tradeoffs in cost and query capability?

---

## Answers

1. Elasticsearch/OpenSearch fully indexes log content (often every field/token) at ingest, producing an inverted index that's frequently several times the size of the raw logs but supports fast arbitrary full-text search. Loki indexes only a small set of labels and stores the raw log body compressed but unindexed — queries narrow to matching label streams first, then scan (grep) the actual content within those streams at query time.
2. Loki is built by Grafana Labs specifically so that teams already comfortable operating Prometheus (label-based thinking, similar service discovery/relabeling patterns, similar storage/compaction operational concerns) can adopt Loki with minimal new conceptual overhead, rather than having to learn a fundamentally different system like Elasticsearch.
3. `|=` keeps lines that contain a literal string (substring match). `|~` keeps lines that match a regular expression — more powerful/flexible but more computationally expensive to evaluate, so `|=` should generally be applied first when both are usable, to narrow the working set before the regex runs.
4. By parsing it out at **query time** using a parser stage like `| json` or `| logfmt`. This extracts fields from the log body into query-scoped fields you can filter/aggregate on, without those fields ever being written into Loki's actual index — trading query-time CPU for zero ingest-time indexing cost.
5. `sum(rate({app="api"} |= "error" [5m]))`
6. `unwrap <field>` extracts a numeric value from a field already pulled out by a parser stage (like `| json`), turning log-derived text into an actual number Loki's range-vector aggregation functions (`quantile_over_time`, `avg_over_time`, `sum_over_time`, etc.) can operate on — without it, those functions have no numeric value to aggregate, since raw log lines are just text.
7. Promtail is in maintenance mode as of Grafana Labs' 2024 announcement, being superseded by **Grafana Alloy**, their unified OpenTelemetry-based collector — though Promtail remains widely deployed and functional in existing production systems.
8. Fluent Bit: extremely lightweight (written in C, low resource footprint) and is the default/pre-installed logging agent on many managed Kubernetes offerings (EKS, GKE) — a strong "boring, proven, already there" choice. Vector: offers VRL (Vector Remap Language), a genuine inline scripting/transform language, useful when your pipeline needs complex conditional reshaping of log data that Fluent Bit's plugin-chain config can't cleanly express, and is generally positioned as higher-throughput under heavy load.
9. Every unique label value combination creates an entirely separate Loki stream; a label like `trace_id` or `user_id` takes effectively unbounded distinct values, creating an unbounded (and ever-growing) number of streams — exactly the cardinality explosion problem Loki's label-only-indexing design was meant to avoid. That value should instead live in the log line's JSON/logfmt body, queryable at query time via `| json`/`| logfmt` rather than as an indexed label.
10. Because each Loki stream is stored as its own compressed chunk, high cardinality doesn't just bloat an index (as in Prometheus) — it also fragments data into many small, poorly-compressed chunks, since compression works better over longer runs of similar data within one stream. This compounds the usual "too many index entries" cost with a storage/compression-efficiency cost specific to Loki's chunk-based storage model.
11. A correlation ID field — typically `trace_id`, ideally propagated via OpenTelemetry context — must appear in every relevant log line as a structured field. Loki's `derivedFields` configuration matches a regex against that field and turns it into a clickable link into the corresponding trace; without the field present in the log content, there's no value for the regex to match and no link can be constructed.
12. Strong answer: "Elasticsearch/OpenSearch fully indexes log content at ingest, which supports fast, arbitrary full-text search and rich aggregations on any field, but at real cost — the index is often several times the raw log size, and that indexing cost is paid on every log line whether it's ever queried or not. Loki only indexes a small set of labels (namespace, app, level, etc.) and stores raw log content compressed but unindexed in object storage, narrowing to relevant streams via labels and then scanning content at query time. That makes Loki significantly cheaper to run for high-volume Kubernetes logging, especially when queries naturally start from label filters — but it trades away fast arbitrary full-text/field search across unstructured content, which is exactly what Elasticsearch's index is built for. In practice, I'd pick Loki when logs are Kubernetes-native and label dimensions are well understood upfront, and I'd pick OpenSearch when the org genuinely needs complex ad-hoc search or aggregation across arbitrary fields — e.g., security/audit log analysis — and can budget for the higher indexing/storage cost that capability requires."
