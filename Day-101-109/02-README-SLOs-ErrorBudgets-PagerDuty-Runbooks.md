# Day 101-109 — Observability Capstone II: SLOs, Error Budgets, PagerDuty, and Runbooks

**Phase:** 3 – Observability | **Week:** W17-W18 | **Domain:** Review | **Flag:** 📌

## Brief

A dashboard full of metrics doesn't tell anyone whether the service is "healthy enough" — that requires a number everyone agreed on in advance: an SLO. This file covers defining SLIs/SLOs for a real service, the error-budget math that turns "99.9% uptime" into an actual operational policy, the multi-window burn-rate alerting technique that avoids both alert fatigue and slow detection, and what happens after an alert fires: PagerDuty routes it to a human, and a runbook tells that human what to do before they've had coffee.

## SLI, SLO, SLA — the actual difference

- **SLI (Service Level Indicator)** — a *measurement*: a specific, quantifiable metric of behavior. Example: "the proportion of HTTP requests to `checkout-service` that complete successfully in under 300ms."
- **SLO (Service Level Objective)** — a *target* for that SLI, usually over a rolling window. Example: "99.9% of requests over a rolling 30-day window meet the SLI above." This is an internal engineering commitment.
- **SLA (Service Level Agreement)** — an SLO with a *contractual consequence* attached (usually to a customer, with a financial penalty for missing it). Not every service needs an SLA; every service you're on-call for should have an SLO.

The interview-critical distinction: **SLOs are internal and set deliberately looser than what you can technically achieve**, to leave room for planned risk (deploys, experiments, migrations) — an SLO of 99.9% is a *ceiling on how much unreliability is acceptable*, not a floor you're trying to exceed for its own sake.

## Choosing good SLIs for a real test app

Two SLI classes cover almost everything:

- **Availability SLI**: `good_events / valid_events`, e.g., `count(http_requests{code!~"5.."}) / count(http_requests{code!~"3.."})` — deliberately excluding client errors (4xx) and redirects from the denominator when they're not the service's fault, but always counting them consistently, decided in advance.
- **Latency SLI**: the proportion of requests under a threshold, e.g., "95% of requests complete in under 300ms" — computed from a Prometheus histogram with `histogram_quantile`, or more precisely as a ratio using bucket counts directly (ratio-based is more mathematically correct for burn-rate math than recomputing a quantile each time — see the CHEATSHEET for the exact query).

For a sample `checkout-service`, a reasonable starting SLO set:
- **Availability**: 99.9% of requests return non-5xx over a rolling 30-day window.
- **Latency**: 95% of requests complete in under 300ms over a rolling 30-day window.

Pick SLIs the *user* would recognize as "the service working," not internal implementation metrics (CPU usage is not an SLI — it's a cause, not a symptom the user experiences).

## Error budgets — the operational payoff of an SLO

If your SLO is 99.9% availability, your **error budget** is the remaining 0.1% — the amount of unreliability you're allowed to spend before you've broken your promise. Over a 30-day window, 0.1% of requests being errors is your budget; once it's exhausted, the standard SRE policy is to **freeze risky changes** (feature launches, non-critical deploys) until the service is back within budget and burn rate settles.

This reframes reliability work from "zero errors is the goal" (impossible and wasteful — chasing five-nines when the business needs three-nines burns engineering time on the wrong problem) to "spend the budget deliberately on things that matter" (deploys, experiments, planned maintenance) rather than accidentally (bugs, capacity failures).

**Error budget formula:**
```
Error budget = 1 - SLO
Budget remaining = Error budget - (errors observed / total requests) over the window
```
Example: SLO = 99.9% → error budget = 0.1%. If your 30-day window has 10,000,000 requests, your budget is 10,000 "allowed" errors. If you've had 6,000 errors so far this window, you've spent 60% of your budget with time still left in the window — that's the signal to slow down risky changes, before you've fully exhausted it.

## Burn rate — the rate you're spending budget, not the total spent

**Burn rate** answers "at the *current* rate of errors, how fast am I consuming my budget relative to what's sustainable for the window?" A burn rate of 1 means you're consuming budget exactly on pace to exhaust it right at the end of the window (sustainable, boring). A burn rate of 10 means you're burning 10x too fast — at this rate you'd exhaust a 30-day budget in 3 days.

```
burn_rate = (rate of errors observed) / (rate allowed by the SLO)
          = error_rate / (1 - SLO)
```

This single formula is why burn-rate alerting works: **a small burn rate over a long window and a huge burn rate over a short window can represent the same total budget consumption** — but they need very different alerting responses (one is a slow leak worth a ticket tomorrow, the other is a fire worth waking someone up right now).

## Multi-window, multi-burn-rate alerting

The naive approach — alert when the SLI drops below the SLO threshold — is bad in both directions: too slow to catch a severe short outage (by the time a 30-day rolling average moves, real damage is already done), and too noisy for brief blips (a 2-minute error spike that self-resolves shouldn't page anyone). Google's SRE Workbook's standard fix is **multi-window, multi-burn-rate alerts**: pair a *long* window (confirms the problem is real, not a blip) with a *short* window (confirms the problem is still happening right now, not already resolved), at multiple burn-rate severities.

A standard 4-alert setup for a 30-day SLO:

| Severity | Long window | Burn rate threshold | Short window (must also match) | Budget consumed if sustained | Action |
|---|---|---|---|---|---|
| Page (critical) | 1h | 14.4x | 5m | 2% of 30-day budget in 1 hour | Page immediately |
| Page (critical) | 6h | 6x | 30m | 5% of 30-day budget in 6 hours | Page immediately |
| Ticket (warning) | 24h | 3x | 2h | 10% of 30-day budget in 1 day | File a ticket, review next business day |
| Ticket (warning) | 3d | 1x | 6h | 10% of budget over 3 days | File a ticket, low urgency |

Requiring **both** windows to breach before firing is what prevents pages for blips that self-heal in under 5 minutes, while still catching genuinely severe short outages fast (a 14.4x burn rate for even 5-10 minutes is worth waking someone up, because at that rate it would exhaust a whole month's budget in about 2 days).

See CHEATSHEET.md for the exact PromQL recording-rule and alerting-rule syntax for this table.

## PagerDuty — routing an alert to a human

Alertmanager doesn't page anyone directly for anything beyond a webhook — it hands off to **PagerDuty** (or another incident-management tool) via a receiver configuration using PagerDuty's Events API v2 integration key. PagerDuty then owns:

- **Escalation policies** — if the primary on-call doesn't acknowledge within N minutes, escalate to a secondary, then to a team lead. This is what prevents "the on-call engineer's phone was on silent" from becoming a full outage with nobody responding.
- **On-call schedules** — rotations (weekly/daily), overrides for vacation/sick days, and time-zone-aware handoffs.
- **Incident urgency** — map your alert `severity` label to PagerDuty urgency (`critical`/`warning` → `high`/`low`), which determines whether it pages immediately or just creates a low-urgency incident visible on next business-day triage.
- **Deduplication** — a `dedup_key` (Alertmanager sends one automatically, keyed off the alert's labels) so the same underlying issue firing repeatedly doesn't spam multiple separate pages; it updates one open incident instead.

## Runbooks — what makes one actually usable at 3am

A good runbook is written for a sleep-deprived engineer who has never touched this specific service, being paged at 3am, with 10 minutes of patience before they either fix it or escalate. That constraint drives the structure:

- **Title and one-line symptom** — matches the alert name exactly, so whoever's paged can search-and-find it instantly (e.g., "checkout-service: high 5xx burn rate").
- **Impact** — who/what is actually affected, and how severe (helps triage: is this "some users see a slow page" or "no one can check out")?
- **Immediate mitigation steps first, root-cause investigation second** — a runbook's job during an incident is to stop the bleeding, not teach the reader the architecture. Rollback/scale/failover commands go at the top, in copy-pasteable form; "here's how to figure out why" goes below, and can wait until things are stable.
- **Exact commands, not descriptions** — `kubectl rollout undo deployment/checkout-service -n prod`, not "roll back the deployment." A tired engineer at 3am should be able to copy-paste, not translate prose into commands under pressure.
- **Escalation path and contacts** — who to page next if this runbook doesn't resolve it, and any dependency team's on-call info.
- **A link back from the alert itself** — the Alertmanager alert's annotation should include a `runbook_url` pointing directly at this doc; an alert with no linked runbook is an alert that will eventually get ignored or mishandled by whoever's unlucky enough to be paged for it.

The single most common runbook failure isn't "no runbook exists" — it's a runbook that's **stale**: written once, never updated as the service changed, so the commands in it no longer match reality. Treat runbooks like code: reviewed on PRs that touch the service they cover, and verified periodically (a "runbook fire drill" — deliberately follow it during a game day, not a real incident, to confirm it still works).

## Points to Remember

- SLI = measurement, SLO = internal target, SLA = SLO with a contractual/financial consequence — know all three cold, they're distinct and frequently confused in interviews.
- Error budget = `1 - SLO`; it's a deliberate allowance for risk (deploys, experiments), not a failure to reach 100%. Exhausting it is a policy trigger (freeze risky changes), not just a vanity metric.
- Burn rate = how fast you're consuming the budget right now relative to a sustainable pace; a burn rate of 14.4 over 1h roughly means "exhaust the 30-day budget in ~2 days" — that's page-worthy.
- Multi-window, multi-burn-rate alerting (long window to confirm it's real + short window to confirm it's still happening) is the standard fix for the tradeoff between fast detection and noise from transient blips.
- PagerDuty owns escalation, on-call scheduling, urgency mapping, and deduplication — Alertmanager's job stops at deciding *whether* to page and *what* to say; PagerDuty's job is making sure the right human actually responds.
- A good runbook leads with copy-pasteable mitigation commands, not architecture explanation, and is linked directly from the alert via `runbook_url`.

## Common Mistakes

- Picking SLIs from what's easy to measure (CPU, memory) instead of what the user actually experiences (successful/fast requests) — internal resource metrics are causes, not SLIs.
- Setting an SLO of 99.99%+ "just to be safe" without checking whether the business/users actually need that, then burning enormous engineering effort chasing reliability nobody asked for instead of shipping features.
- Alerting directly on "SLI dropped below SLO" with a single window — either misses fast severe outages (long window moves too slowly) or pages for noise (short window alone catches blips that self-resolve).
- Treating error-budget exhaustion as just a number on a dashboard instead of an actual policy trigger — if nothing changes (no freeze on risky launches) when the budget hits zero, the SLO isn't really operating as intended.
- Writing a runbook once at launch and never touching it again — after a few quarters of changes to the service, a stale runbook's commands can be actively wrong, which is worse than no runbook (false confidence during an incident).
- Not mapping alert severity to PagerDuty urgency correctly — a "warning" ticket-level alert misconfigured to page like a critical one trains on-call engineers to ignore pages, which is how a real critical page eventually gets missed too.
