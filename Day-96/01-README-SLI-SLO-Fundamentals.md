# Day 96 — SLOs, SLAs & Error Budgets: SLI & SLO Fundamentals

**Phase:** 3 – Observability | **Week:** W16 | **Domain:** SRE | **Flag:** ⚡ Interview-critical

## Brief

This is the conceptual core of Google's SRE discipline, and arguably the single most interview-tested SRE topic after "what is an error budget." Every reliability decision an SRE team makes — whether to slow down releases, whether an alert should page a human at 3am, whether a feature is "done" — should trace back to a **Service Level Indicator (SLI)** and a **Service Level Objective (SLO)**. Without this framework, reliability work degenerates into "make everything as reliable as possible," which is both impossible and wasteful (100% reliability is not just expensive, it's often the wrong target — see error budgets in file 2). This file builds the vocabulary; file 2 covers error budgets and burn-rate alerting; file 3 covers SLAs and the alerting philosophy shift SLOs enable.

## SLI — Service Level Indicator: what you actually measure

An SLI is a **quantitative measure of some aspect of the level of service provided**, expressed as a ratio or a percentile, always tied to a specific window of time. It's a *measurement*, not a target — the target comes later (that's the SLO).

The canonical SLI categories for a request-driven service:

- **Availability** — the proportion of requests that were served successfully. Usually expressed as `good_events / total_events` over a rolling window, e.g. `(requests_total - requests_5xx) / requests_total`.
- **Latency** — the proportion of requests served faster than a threshold. E.g., "proportion of requests completed in <300ms," expressed as a percentile-based SLI, not an average (same reasoning as Day 95's RED metrics note — averages hide the tail that actually hurts users).
- **Throughput** — requests processed per unit time; less commonly an SLO target on its own (it's usually a capacity/scaling signal), but still tracked as an SLI where "can the system keep up with demand" matters (e.g., batch/data pipelines).
- **Error rate** — proportion of requests resulting in an error, often folded into the availability SLI's definition of "good" vs. "bad" event, but sometimes tracked separately when "available but erroring on a specific feature" needs its own visibility.

**The critical design decision: define "good event" precisely.** A vague SLI ("the service should be up") is useless — you need an unambiguous, queryable definition. A concrete, real example:

```promql
# Availability SLI over a rolling 30-day window (Prometheus)
sum(rate(http_requests_total{job="checkout-api", code!~"5.."}[30d]))
/
sum(rate(http_requests_total{job="checkout-api"}[30d]))
```

```promql
# Latency SLI: proportion of requests under 300ms
sum(rate(http_request_duration_seconds_bucket{job="checkout-api", le="0.3"}[30d]))
/
sum(rate(http_request_duration_seconds_count{job="checkout-api"}[30d]))
```

Note the latency SLI query depends on having a **histogram** metric with a bucket boundary at (or near) your actual threshold — if your histogram buckets are `[0.1, 0.5, 1, 5]` and your SLO target is "under 300ms," you have no bucket boundary at 0.3s and can't compute this precisely; you'd need to either re-bucket the histogram or accept an approximation. This is a very real, very common practical trap — **decide your SLO thresholds before you finalize your histogram bucket boundaries**, not after.

### Choosing what to measure: the user's perspective, not the server's

The single biggest SLI design mistake is measuring what's *easy to measure server-side* instead of what actually reflects **user-perceived** quality. A server can return HTTP 200 while the response body contains a client-visible error, or while it took so long the user already gave up and closed the tab (a request the server logs as "successful" but the user experienced as a failure). Best practice: measure as close to the user as feasible (client-side beacons, real user monitoring, load balancer/edge logs) rather than purely from application-internal metrics, and treat "the user's experience" as the source of truth for what "good" means — even if that's harder to instrument than a simple server error code.

## SLO — Service Level Objective: the target

An SLO is a **target value or range for an SLI, measured over a defined time window**. Example: "99.9% of `checkout-api` requests over a rolling 30-day window will return a non-5xx response." The SLO is a **choice**, not a law of physics — it should be set based on what users actually need and what the business can tolerate, not an arbitrary round number, and definitely not "as high as technically achievable."

Key properties of a well-formed SLO:
- **Time window matters as much as the percentage.** "99.9% over 30 days" and "99.9% over 7 days" are very different commitments — the shorter window is stricter because there's less time to average out a bad incident. Rolling windows (continuously sliding) are generally preferred over calendar-aligned windows (e.g., strict calendar months) because they avoid the "reliability resets to 100% on the 1st of the month" artifact and give a more honest, continuously-current picture.
- **Should be set slightly below what you can realistically achieve, not at your historical best.** If your service has organically run at 99.95% for the last year, setting the SLO at 99.99% guarantees you're perpetually in violation for reasons unrelated to real user impact. The point of an SLO is to be a meaningful, achievable line that separates "acceptable" from "needs attention" — not an aspirational stretch goal.
- **Should reflect what users actually notice.** Going from 99.9% to 99.99% is an order-of-magnitude harder engineering problem (see error budget math in file 2) for a difference users may not perceive at all for many product types — internal tools rarely need 99.99%; payment-critical or safety-critical paths might genuinely need it. This tradeoff, explicitly made and written down, is the entire point of the exercise.

## Points to Remember

- SLI = the measurement (a ratio/percentile of "good" events over "total" events, over a time window). SLO = the target for that measurement. Neither exists without the other — an SLO with no clearly-defined SLI is just a vague promise.
- Measure SLIs as close to actual user experience as possible — server-side "success" (HTTP 200) is not the same thing as user-perceived success.
- Latency SLIs must be percentile-based, and your histogram bucket boundaries must actually include the threshold you care about, or the SLI query can't be computed precisely.
- Rolling windows (continuously sliding, e.g. "trailing 30 days") are generally preferred over calendar-aligned windows because they avoid an artificial reset and give a continuously honest picture.
- SLOs should be set at a realistic, meaningful line — not your historical best-case, and not "as high as possible" — because tighter SLOs are exponentially more expensive to sustain (error budget math, file 2).

## Common Mistakes

- Defining an SLI in terms of server-side HTTP status codes alone, missing cases where the server returns 200 but the client-visible experience was actually a failure (slow enough to abandon, or a degraded/partial response).
- Setting a latency SLO threshold that doesn't correspond to any actual histogram bucket boundary, making the SLI mathematically impossible to compute precisely from existing metrics — discovered only after the SLO is already published and stakeholders expect a dashboard.
- Copying "99.99% because that's what serious companies do" without doing the math on what that actually costs in engineering effort/error budget (file 2) or whether users would even notice the difference from 99.9%.
- Using calendar-month windows and being misled by the "reset to 100% on the 1st" artifact — a bad incident on the 30th barely dents that month's number, while the exact same incident on the 2nd defines the entire month.
- Treating throughput/capacity metrics as if they were reliability SLOs on their own, rather than recognizing them as capacity-planning signals that feed into (but aren't identical to) availability/latency SLOs.
