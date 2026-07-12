# Day 95 — Jaeger & Tempo: Practical Trace Analysis — N+1 Queries & RED Metrics

**Phase:** 3 – Observability | **Week:** W16 | **Domain:** Tracing

## Brief

Having a tracing backend is only useful if you know how to *read* a trace to diagnose a real problem. This file covers the two most practically valuable skills for day-to-day and on-call work: spotting the N+1 query anti-pattern in a trace waterfall, and understanding RED metrics (Rate, Errors, Duration) — both as a framework for what to alert on, and as something Tempo can derive automatically from traces. This is also a direct rehearsal for today's interview question about diagnosing high p99 latency via tracing.

## Finding N+1 queries via tracing

The **N+1 query problem**: code that, to render one logical result (say, a list of N orders with their customer names), executes 1 query to fetch the list, then N additional queries — one per row — to fetch related data, instead of a single join or a single batched query. It's one of the most common real-world causes of "this endpoint got slow after we added a feature" and is often invisible in code review because each individual query looks innocuous.

**What it looks like in a trace waterfall:** instead of the expected small number of DB spans, you see a long, repetitive run of near-identical sibling spans — same operation name (`SELECT customer FROM customers WHERE id = ?`), same short duration each, but *dozens or hundreds of them*, stacked one after another (often sequentially, not even in parallel) under the same parent span. Visually this is unmistakable: a "staircase" or "comb" pattern of many thin, same-width bars, cumulatively consuming far more wall-clock time than the rest of the request combined, even though no single query is slow.

**Why this is easy to miss without tracing:** a metrics dashboard shows "DB query p99 = 12ms" (true! each individual query genuinely is fast) — completely hiding that the *endpoint* is slow because it makes 200 of them serially. A log file shows 200 lines of successful, fast queries with no obvious error. Only the trace, showing the actual count and cumulative time attributable to a single parent span, exposes the pattern.

**Diagnosis workflow in Jaeger/Tempo:**
1. Sort/filter traces for the slow endpoint by duration (Tempo: `{ resource.service.name = "orders-api" && duration > 500ms }`).
2. Open the slowest trace, look at the waterfall — count repeated span operation names under one parent.
3. Confirm the pattern is proportional to input size (e.g., is span-count roughly equal to `len(orders)`?) — if request size N correlates with DB-span count N, that's the smoking gun.
4. Fix is almost always: batch the lookup (`WHERE id IN (...)`), add a join, or use the ORM's eager-loading/prefetch feature (e.g., Django's `select_related`/`prefetch_related`, SQLAlchemy's `joinedload`) instead of lazy-loading each related object individually.

## RED metrics — and where they come from

**RED** is a metrics framework (coined by Weaveworks' Tom Wilkie, a sibling to Google's older **USE** method for resources) specifically for request-driven services:

- **R**ate — requests per second the service is handling.
- **E**rrors — the rate/fraction of those requests that are failing.
- **D**uration — how long requests take (usually as a distribution: p50/p95/p99, not just an average — see Day 94's histogram note on why averages of averages are meaningless).

RED is deliberately minimal and per-service/per-endpoint — it's the "is this service healthy" dashboard you'd want on every single service in your fleet, consistently, so an on-call engineer can pull up *any* service's dashboard and immediately answer "is it serving traffic normally, is it erroring, is it slow" without needing service-specific custom dashboards.

**Deriving RED metrics from traces instead of hand-instrumenting them:** every span already carries a duration, a status (OK/Error), and identifying attributes (service name, operation name) — which is *exactly* the raw material RED needs. Tempo's **metrics-generator** component (and similarly, the OTel Collector's `spanmetrics` connector/processor) consumes the span stream and aggregates it into Prometheus-style RED metrics automatically:

```yaml
# OTel Collector spanmetrics connector (conceptual config)
connectors:
  spanmetrics:
    histogram:
      explicit:
        buckets: [2ms, 8ms, 50ms, 100ms, 200ms, 500ms, 1s, 5s]
    dimensions:
      - name: http.method
      - name: http.status_code

service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [spanmetrics, otlp/tempo]   # traces flow to storage AND to the metrics generator
    metrics:
      receivers: [spanmetrics]                # generated RED metrics flow onward
      exporters: [prometheus]
```

This means you get consistent, uniform RED dashboards across every service **for free**, purely as a byproduct of tracing instrumentation you already did — no separate hand-written `http_requests_total`/`http_request_duration_seconds` metrics needed (though many teams still keep both, since span-derived metrics inherit sampling: if you're tail-sampling traces at 10%, span-derived RED metrics can undercount unless generated *before* the sampling processor in the pipeline).

## Walking through the interview question: diagnosing high p99 latency

**The question:** *A microservice has high p99 latency. Walk me through using distributed tracing to find the root cause.*

**A strong answer, step by step:**
1. **Confirm scope with RED metrics first.** Check the service's RED dashboard — is p99 latency actually elevated for *this* service specifically, or is it a downstream dependency dragging it up? Check error rate too — sometimes "latency" is actually retries-after-timeouts inflating the tail.
2. **Pull representative slow traces, not just one.** Query the tracing backend for traces of this specific service/endpoint above the p99 threshold, over the affected time window (Tempo: `{ resource.service.name = "X" && duration > <p99 threshold> }`). Look at several, not just one — one anecdote could be a fluke; a pattern across many confirms a systemic cause.
3. **Read the waterfall for where time concentrates.** Is one specific child span (a DB call, a downstream service call, a cache miss path) consistently the largest contributor across these slow traces? That's your candidate root cause span.
4. **Check for the N+1 pattern specifically** if the slow contributor is many small repeated spans rather than one big one (see above).
5. **Correlate to logs at that span** for the precise error/detail (e.g., a specific slow query's actual SQL, a downstream 503, a lock wait) — this is the trace-to-logs jump from file 2.
6. **Check whether it's a downstream dependency, not your own service** — if the slow span is a call to another team's service, the trace proves it and gives you the evidence to route the investigation (and the incident, if one's open) to the right owner instead of everyone guessing.
7. **Correlate with deploys/config changes** around when p99 started rising — traces tell you *where* the time goes structurally, but a recent deploy/feature flag/config change is often *why* it changed. Check whether the newly-slow span corresponds to a recently shipped code path.

This structure — metrics to confirm/scope, traces to localize, logs to get exact detail, then correlate with recent changes — is the generalized SRE debugging loop, and demonstrating it (rather than jumping straight to "I'd look at the trace") is what separates a strong interview answer from a shallow one.

## Points to Remember

- N+1 queries show up in a trace as many same-named, similarly-short sibling spans under one parent, cumulatively dominating the request's duration — invisible in per-query metrics/logs, obvious in a waterfall.
- RED = Rate, Errors, Duration (duration as percentiles, not averages) — the standard, minimal per-service health dashboard for request-driven systems.
- RED metrics can be generated automatically from spans (Tempo metrics-generator / OTel Collector `spanmetrics` connector) instead of hand-instrumented — but watch for sampling interactions if generated after a tail-sampling processor.
- The diagnostic order for "why is p99 high": RED metrics to confirm scope → representative slow traces to localize the bottleneck span → logs at that span for exact detail → correlate with recent deploys/config changes.
- Look at *multiple* slow traces, not one, before concluding a root cause — one trace can be an outlier/fluke.

## Common Mistakes

- Diagnosing latency from a single slow trace and generalizing to "the whole system is slow because of X," when that one trace was an outlier (e.g., a one-off GC pause) unrelated to the systemic p99 issue.
- Spotting many DB spans in a trace and assuming "the database is slow" instead of checking whether it's actually an N+1 pattern where each query is fast but there are far too many of them — the fix (batch/join/eager-load) is completely different from a genuinely slow query needing an index.
- Averaging per-host p99s together to get a "fleet p99" — percentiles don't average meaningfully; you need the raw distribution (histogram) aggregated centrally, same trap as Day 94's metrics note.
- Generating RED metrics from spans *after* a tail-sampling processor that drops 90% of traces, then wondering why the derived request-rate metric undercounts reality by 10x — span-metrics generation should happen before sampling, or you need to account for the sampling ratio.
- Treating traces as a replacement for logs/metrics rather than a complement — some issues (a memory leak growing over hours, a slow gradual disk fill) are invisible in any single trace and only show up as a metric trend over time.
