# Day 92 — Loki & Log Aggregation: LogQL Basics

**Phase:** 3 – Observability | **Week:** W15 | **Domain:** Logging | **Flag:** ⚡ Interview-critical

## Brief

LogQL is Loki's query language, deliberately built to feel like PromQL's cousin — if Day 89's PromQL clicked for you, LogQL will feel immediately familiar in structure, with the key difference being that a LogQL query starts from a **label selector** (exactly like PromQL) but then adds a **log-content filter pipeline** on top, since the whole point of Loki is that log bodies aren't pre-indexed the way metric labels are.

## Anatomy of a LogQL query

Every LogQL query starts with a **log stream selector** — syntactically identical to a PromQL label matcher:

```logql
{namespace="checkout", app="api", level="error"}
```

This alone returns every raw log line from streams matching those labels — the equivalent of `tail -f` scoped to exactly the pods/labels you care about. From here, you chain **filter expressions** and **parsers**:

```logql
{namespace="checkout", app="api"} |= "timeout"
```

`|=` is a **line-contains filter** — keep only lines containing the literal string `"timeout"`. This is the grep-at-query-time step described in the previous file: Loki narrows to the matching *streams* using the label index, then scans line content within those streams for the literal/regex match.

### Line filter operators

```logql
{app="api"} |= "error"        # line contains "error"
{app="api"} != "healthcheck"   # line does NOT contain "healthcheck"
{app="api"} |~ "err(or)?"      # line matches regex
{app="api"} !~ "debug|trace"   # line does NOT match regex
```

You can chain multiple filters, and Loki evaluates them left-to-right, narrowing the result set at each step — put your **cheapest, most selective filter first** (usually the label selector already did the heavy lifting; among line filters, a literal `|=` string match is cheaper than a `|~` regex, so filter with `|=` before adding a `|~` refinement).

### Parsers — turning unstructured text into fields you can query

If your logs are structured (JSON, logfmt), LogQL can **parse them into labels at query time** without needing them indexed ahead of time:

```logql
{app="api"} | json
{app="api"} | json | status_code = "500"
{app="api"} | logfmt | duration > 500ms
```

`| json` parses each line as JSON and extracts every top-level field as a queryable label *for that query only* (not written back into Loki's index) — so `status_code`, `duration`, `trace_id`, whatever your app logs as JSON keys, become filterable and usable in aggregations, without you having pre-declared them as indexed labels. `| logfmt` does the same for `key=value key2=value2` formatted lines (a common structured-but-not-JSON convention, notably used by HashiCorp tools and many Go services).

This is the key mechanism that resolves an apparent contradiction: "Loki doesn't index log content, so how do I filter/aggregate on a specific field inside the log body?" — the answer is parsing it out **at query time** via `| json` / `| logfmt` / `| pattern` / `| regexp`, trading a bit of query-time CPU for zero ingest-time indexing cost.

### Metric queries — turning logs into numbers

LogQL can also produce **numeric time series** from log data, using the same range-vector syntax PromQL uses for `rate()`:

```logql
# count of matching log lines per second, over 5m windows
sum(rate({app="api"} |= "error" [5m]))

# count grouped by a parsed field
sum by (status_code) (count_over_time({app="api"} | json | __error__="" [5m]))

# extract a numeric field and compute a quantile over it
quantile_over_time(0.99, {app="api"} | json | unwrap duration [5m])
```

- `rate(...[5m])` on a log stream = log lines per second matching the filter — this is exactly how you'd alert on "error log rate spiking" the same way you'd alert on a PromQL error-rate query.
- `count_over_time(...[5m])` = total matching lines in the window (the `increase()`-equivalent for logs).
- `unwrap <field>` extracts a **numeric value** from a parsed field (e.g., a `duration` field pulled out via `| json`) so you can run `quantile_over_time()`, `avg_over_time()`, `sum_over_time()`, etc. directly on values embedded in log lines — effectively deriving histogram-like percentiles from unstructured/semi-structured log data when you don't have a proper Prometheus histogram instrumented.

## Points to Remember

- Every LogQL query starts with a label stream selector, identical in syntax to PromQL — this is what narrows to specific streams before any content scanning happens.
- Line filters: `|=` (contains), `!=` (not contains), `|~` (regex match), `!~` (regex not match) — chain cheapest/most selective first.
- `| json` / `| logfmt` parse structured log content into query-time-only fields, letting you filter/aggregate on fields that were never pre-indexed — this is how Loki reconciles "no full-text indexing" with "still queryable by field."
- `rate()`/`count_over_time()`/`quantile_over_time()` turn log streams into numeric time series, directly usable in Grafana panels or Grafana-managed alerts the same way PromQL output is.
- `unwrap <field>` pulls a numeric value out of a parsed log field so range-vector aggregation functions can operate on it.

## Common Mistakes

- Writing a LogQL query with an overly broad label selector (e.g., `{namespace=~".+"}`) and relying on line filters to do all the narrowing — this defeats the entire performance model, since Loki's speed comes from labels narrowing the stream set *before* any content scan.
- Putting an expensive regex filter (`|~`) before a cheap literal filter (`|=`) in the chain — reorder so the cheap, highly selective filter runs first and reduces the working set before the regex has to evaluate against it.
- Forgetting that `| json`/`| logfmt`-parsed fields are query-time only, not stored as indexed labels — you cannot use them in the initial stream selector `{...}`, only in filter/aggregation stages after the parser runs.
- Assuming `| json` gracefully skips malformed JSON lines silently everywhere without checking `__error__` — mixed structured/unstructured logs in the same stream can produce parser errors that need explicit filtering (`| __error__=""`) to exclude from aggregations.
- Trying to run PromQL-style instant-vector-only functions (like plain `sum()`) directly on a raw log selector without a range vector — metric queries in LogQL still require the same `func(...[range])` shape as PromQL; forgetting the range duration is a frequent syntax error for people coming fresh from PromQL.
