# Day 97 — Incident Management: Runbooks & Playbooks

**Phase:** 3 – Observability | **Week:** W16 | **Domain:** SRE | **Flag:** ⚡ Interview-critical

## Brief

A runbook is what turns "the person who built this system knows how to fix it" into "whoever is on-call this week can fix it, even at 3am, half-asleep, under pressure, without waking up the original author." Without runbooks, institutional knowledge is trapped in specific people's heads, which is both a bus-factor risk and a guarantee that incidents take longer to resolve whenever the "right" person isn't immediately reachable. Today's hands-on activity is literally writing one for a real failure mode — this file is what makes that exercise produce something actually useful instead of generic filler.

## Runbook vs. playbook — the (loose but useful) distinction

- **Runbook** — a specific, mechanical, step-by-step procedure for a *known* failure mode or routine operational task: "disk usage alert on `db-primary`," "restart the stuck Kafka consumer group," "rotate the expiring TLS cert." Written to be followed nearly verbatim, minimal judgment required.
- **Playbook** — a broader strategic guide for a *category* of situation, involving more judgment calls: "how to run any P1 incident," "how to handle a security breach," "how to communicate during a data-loss event." Less "run this exact command" and more "here's the decision framework and checklist."

In practice teams use the terms loosely/interchangeably, but the underlying distinction (mechanical steps for a known problem vs. a judgment framework for a category of problem) is worth being precise about in an interview answer.

## What makes a runbook actually get used under pressure

A runbook that looks thorough in a design doc review often turns out useless during a real incident because it was written for calm, unhurried reading — not for someone stressed, sleep-deprived, and time-pressured at 3am. The properties that separate a runbook that gets followed from one that gets abandoned mid-incident:

- **Exact, copy-pasteable commands — not descriptions of commands.** "Check if the consumer group is lagging" is useless under pressure; `kafka-consumer-groups.sh --bootstrap-server $BROKER --describe --group checkout-consumers` is usable. Every command should be runnable exactly as written, with placeholders clearly marked (`$BROKER`, `<POD_NAME>`) and, ideally, a one-line note on how to obtain the placeholder value.
- **A clear, unambiguous trigger condition at the top.** "Use this runbook when: PagerDuty alert `CheckoutAPIHighErrorRate` fires AND Grafana dashboard X shows Y." Someone should be able to confirm in 10 seconds whether this is even the right runbook for what they're seeing, before investing time following it.
- **Diagnosis steps before remediation steps**, each with an expected output described, so the responder can confirm at each stage whether they're actually on the right track — not just blindly executing a sequence of commands and hoping. E.g., "run X — if you see Y, this confirms Z; proceed to step 4. If you see W instead, this is a different failure mode, see runbook B."
- **An explicit rollback/escalation path if the runbook doesn't work.** Every runbook should account for "I followed all of this and the problem persists" — what next, who do you page, what's the fallback (e.g., "roll back to the last known-good deploy," "fail over to the DR region").
- **Kept current, and treated as a first-class artifact, not documentation debt.** A stale runbook (referencing a decommissioned host, an old command syntax, a dashboard that's been renamed) actively wastes time during an incident and erodes trust — once responders get burned by a stale runbook once, they stop trusting/using runbooks at all, which is worse than not having them. The fix: update the runbook as part of the postmortem process (file 4) whenever an incident reveals it was wrong/incomplete/missing, and periodically audit runbooks even absent an incident.
- **Linked directly from the alert itself.** The best practical pattern: the PagerDuty/alert payload includes a direct link to the relevant runbook, so the responder doesn't have to search a wiki under pressure — the alert *is* the entry point to the fix procedure.

## A concrete runbook example (structure to model your lab exercise on)

```markdown
# Runbook: Checkout API — Elevated 5xx Error Rate

**Trigger:** PagerDuty alert `CheckoutAPIHighErrorRate` (fast-burn, page-severity) fires.

## 1. Confirm scope (2 min)
- Open Grafana dashboard "Checkout API - RED" — confirm error rate and which endpoint(s).
- Check if this correlates with a recent deploy: `kubectl rollout history deployment/checkout-api -n prod`
- Expected: error rate graph shows a step-change coinciding with a deploy timestamp, OR a gradual climb (different failure mode — see step 3b).

## 2. If correlated with a recent deploy -> rollback
```bash
kubectl rollout undo deployment/checkout-api -n prod
kubectl rollout status deployment/checkout-api -n prod --timeout=120s
```
- Confirm error rate returns to baseline within 5 minutes on the RED dashboard.
- If resolved: proceed to step 4 (verify + communicate). If not: proceed to step 3b.

## 3b. If NOT correlated with a deploy -> check downstream dependency
- Check payment-gateway's own status dashboard/health endpoint: `curl -s https://payment-gateway.internal/healthz`
- Check for connection pool exhaustion: Grafana panel "checkout-api DB connection pool utilization"
- If payment-gateway is unhealthy: this is their incident too — page/notify #payment-gateway-oncall, link this incident.
- If connection pool exhausted: see runbook "DB Connection Pool Exhaustion" (link).

## 4. Verify and communicate
- Confirm RED dashboard back to baseline for 10 continuous minutes before declaring resolved.
- Post resolution update in #incidents and update status page if one was opened.

## Escalation
If none of the above resolves it within 20 minutes, escalate to checkout-team EM and open a P1 war room.
```

Note the structure: trigger condition, diagnosis-with-expected-output, branching paths for different root causes, explicit commands, and an escalation fallback — this is the template to follow for today's lab.

## Points to Remember

- Runbook = mechanical steps for a known failure mode; playbook = judgment framework for a category of situation — teams use the terms loosely, but know the distinction.
- A usable runbook has copy-pasteable exact commands, a clear trigger condition, diagnosis-before-remediation with expected outputs at each step, and an explicit escalation/rollback path if it doesn't work.
- Runbooks decay — treat staleness as a bug, fix it as part of the postmortem process every time an incident reveals a gap, and audit periodically even without an incident.
- The best runbooks are linked directly from the alert itself, so there's zero search time between "paged" and "following the fix procedure."
- The purpose of a runbook is bus-factor reduction — it lets *anyone* on-call resolve a known issue competently, not just the original system author.

## Common Mistakes

- Writing a runbook as prose description ("check the logs for errors") instead of exact, runnable commands — forces the stressed 3am responder to improvise the actual command syntax, which is exactly what a runbook exists to avoid.
- Never updating a runbook after an incident reveals it was wrong or incomplete — the next responder hits the same stale, misleading instructions and loses trust in runbooks generally.
- Writing a runbook with only a "happy path" (assumes the fix works) and no fallback/escalation path for when it doesn't — leaves the responder stuck with no next step exactly when they most need one.
- Burying runbooks in a wiki with no link from the alert itself, so under time pressure the responder has to search for the right document instead of clicking straight through from the page.
- Treating runbook-writing as a one-time task instead of an ongoing practice tied to the postmortem process — new failure modes discovered in incidents never get turned into new runbooks, so the same category of outage keeps requiring improvisation.
