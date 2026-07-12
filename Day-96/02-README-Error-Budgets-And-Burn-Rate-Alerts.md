# Day 96 — SLOs, SLAs & Error Budgets: Error Budgets & Burn Rate Alerts

**Phase:** 3 – Observability | **Week:** W16 | **Domain:** SRE | **Flag:** ⚡ Interview-critical

## Brief

The error budget is the single idea that turns SLOs from a passive reporting number into an active decision-making tool. It reframes "reliability" from an unbounded goal ("more reliable is always better") into a **finite, spendable resource** that can be traded off explicitly against feature velocity. This is exactly what today's interview question is asking about, and it's the most commonly cited concrete mechanism from the Google SRE book in real interviews — expect to both define it and describe how you'd operationalize it.

## What an error budget actually is

If your SLO is "99.9% availability over 30 days," then by definition **0.1% of requests are allowed to fail** without violating the SLO. That 0.1% *is* the error budget — the amount of "unreliability" you're explicitly permitted to spend before you've broken your promise to users.

Converting the percentage into a wall-clock time budget makes it concrete and easy to reason about, so this is the standard framing:

| SLO target | Allowed downtime / month (30 days) | Allowed downtime / week | Allowed downtime / year |
|---|---|---|---|
| 99% | 7h 18m | 1h 41m | 3d 15h |
| 99.9% | **43.8 minutes** | 10.1 minutes | 8h 45m |
| 99.95% | 21.9 minutes | 5.04 minutes | 4h 22m |
| 99.99% | 4.38 minutes | 1.01 minutes | 52.6 minutes |
| 99.999% | 26.3 seconds | 6.05 seconds | 5.26 minutes |

The 99.9%-over-30-days row (43.8 minutes) is the canonical example everyone quotes — know it cold, and know *how* to derive it: `30 days × 24h × 60min × (1 - 0.999) = 43,200 × 0.001 = 43.2` minutes (the commonly cited 43.8 figure uses 30.44 average days/month; either derivation is fine as long as you can show the math live). Note the jump between 99.9% and 99.99% is a **10x reduction** in allowed downtime for one more "nine" — this is why each additional nine is exponentially, not linearly, more expensive to engineer for (redundancy, failover automation, chaos testing, on-call rigor all have to improve substantially to sustain the tighter budget).

## Why the budget matters: it's a decision-making tool, not just a number

The entire point of expressing reliability as a *budget* rather than a *target to always beat* is that it gives teams a principled, pre-agreed answer to the perpetual tension between shipping features and being careful:

- **If the error budget is not yet spent** (the service is comfortably within its SLO for the window), the team has room to take risks: ship faster, deploy more frequently, run experiments, do a risky migration — because there's budget to absorb the occasional incident this causes.
- **If the error budget is exhausted** (or dangerously close), the pre-agreed policy kicks in: freeze non-critical feature releases, redirect engineering effort to reliability work (fixing root causes, adding tests, hardening infra), and treat further risk-taking as against the agreed policy until the budget recovers (rolls back into the window).

This is a **negotiated, written-down policy** (often called an "error budget policy") agreed between product/eng leadership *before* an incident happens — not an ad-hoc argument during a postmortem about whether to freeze releases. Having it pre-agreed removes politics from the decision: the data (budget remaining) makes the call, not whoever argues loudest in the moment.

## Burn rate: the derivative of the error budget

**Burn rate** is the rate at which you're consuming your error budget, relative to a sustainable pace. A burn rate of **1** means you're consuming the budget exactly as fast as the SLO window allows (i.e., you'll land exactly at your SLO target by the end of the window — cutting it exactly to the line). A burn rate of **10** means you're consuming budget 10x faster than sustainable — at that rate, a 30-day budget would be fully exhausted in 3 days.

**Why burn rate — not raw "error budget remaining" — is what you alert on:** if you only alert "error budget is exhausted," you find out *after* you've already violated the SLO — too late to react. Burn rate lets you detect "at this rate, we're on track to blow the budget" *before* it happens, with enough lead time to actually respond.

### Multi-window, multi-burn-rate alerting (the Google SRE-recommended pattern)

A single burn-rate threshold has an unavoidable tradeoff: a short lookback window (e.g., 5 minutes) detects fast, severe outages quickly, but is noisy (false positives from brief blips) and doesn't catch slow, sustained degradations. A long lookback window (e.g., 24 hours) is stable and catches slow burns, but takes 24 hours to notice a sudden total outage — far too slow to page anyone.

The standard solution is **multiple simultaneous burn-rate alerts, at different severities**, each using a **short window + a longer window** together (the longer window prevents a brief spike from firing the alert if it doesn't sustain, cutting false positives) — this is the design in the Google SRE Workbook's chapter on alerting, and it's what tools like **Sloth** generate as boilerplate Prometheus rules:

```yaml
# Sloth-style SLO burn-rate alert examples (conceptual, PromQL-based)
# Fast burn: page immediately — would exhaust a 30-day budget in ~2 days
- alert: CheckoutAPIErrorBudgetBurnFast
  expr: |
    (
      sum(rate(http_requests_total{job="checkout-api", code=~"5.."}[1h]))
      /
      sum(rate(http_requests_total{job="checkout-api"}[1h]))
    ) > (14.4 * 0.001)
    and
    (
      sum(rate(http_requests_total{job="checkout-api", code=~"5.."}[5m]))
      /
      sum(rate(http_requests_total{job="checkout-api"}[5m]))
    ) > (14.4 * 0.001)
  labels: {severity: page}

# Slow burn: ticket, not a page — would exhaust the budget in ~a week if sustained
- alert: CheckoutAPIErrorBudgetBurnSlow
  expr: |
    (
      sum(rate(http_requests_total{job="checkout-api", code=~"5.."}[6h]))
      /
      sum(rate(http_requests_total{job="checkout-api"}[6h]))
    ) > (3 * 0.001)
  labels: {severity: ticket}
```

The `14.4` and `3` multipliers come from the standard Google SRE Workbook table pairing burn rate, long window, and the fraction of budget consumed — e.g., a burn rate of 14.4x sustained for 1 hour consumes 2% of a 30-day budget, which is judged severe enough to page immediately; a burn rate of 3x sustained for 6 hours also consumes about 2%, but at a pace that only warrants a ticket, not a page, because there's more time to respond before real damage. Each severity tier pairs a **short window** (fast detection) with a **long window** (confirms it's sustained, not a blip) using the *same* threshold multiplier on both, and only fires the alert when both agree.

## Points to Remember

- Error budget = `1 - SLO target`, expressed as either a fraction of requests or a wall-clock time over the SLO's window. Memorize: 99.9% over 30 days ≈ 43.8 minutes of allowed "badness."
- Each additional nine of reliability cuts the allowed downtime by 10x — reliability improvements get exponentially more expensive, not linearly.
- An error budget's power is as a **pre-agreed decision rule**: budget available → ship fast and take risks; budget exhausted → freeze features, focus on reliability — decided by policy, not argued case-by-case during an incident.
- Burn rate = speed of budget consumption relative to sustainable pace; alert on burn rate (leading indicator), not just "budget already exhausted" (lagging indicator, too late).
- Multi-window, multi-burn-rate alerts (short window for fast detection + long window to confirm sustained, at multiple severity tiers) balance speed of detection against false-positive noise — this is what Sloth generates automatically from an SLO spec.

## Common Mistakes

- Treating the error budget purely as a reporting metric ("we're at 99.92% this month, neat") instead of an actual, enforced decision rule that changes team behavior when it's low.
- Alerting only on "SLO violated" (budget fully consumed) rather than on burn rate — by the time budget is fully gone, the damage (and the user impact) has already happened; there was no early warning.
- Using a single-window burn-rate alert (e.g., just "high error rate over 5 minutes") — either too noisy (pages on transient blips) or too slow (misses fast severe outages if the window chosen is long), because one window can't serve both goals at once.
- Not having a written, leadership-agreed error budget *policy* before the first time the budget is exhausted — leads to an ad-hoc, political argument over whether to actually freeze releases, exactly when tempers are highest (mid-incident or post-incident).
- Confusing "burn rate 1" with "everything is fine" — burn rate 1 means you're on pace to land *exactly* at your SLO threshold, with zero margin; many teams treat sustained burn rate ≥1 as a signal to start caring, not wait for it to exceed 1 significantly.
