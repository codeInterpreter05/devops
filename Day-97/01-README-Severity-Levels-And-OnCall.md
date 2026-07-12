# Day 97 — Incident Management: Severity Levels & On-Call Fundamentals

**Phase:** 3 – Observability | **Week:** W16 | **Domain:** SRE | **Flag:** ⚡ Interview-critical

## Brief

Everything else in incident management — escalation policies, runbooks, postmortems — hangs off two upstream decisions: how severe is this, and whose job is it to respond right now. Get severity classification wrong and you either page a VP for a typo in a marketing page or let a payment-processing outage sit in a ticket queue for two hours. Get on-call structure wrong and burnout follows within a quarter. This is squarely in "have you actually been on-call" interview territory — expect follow-ups that probe whether your answer is textbook or lived experience.

This day is split into four focused files:

1. **This file** — incident severity levels (P1-P4) and on-call fundamentals.
2. **[02-README-Escalation-And-Communication.md](02-README-Escalation-And-Communication.md)** — PagerDuty-style escalation policies and communication discipline during an incident.
3. **[03-README-Runbooks-And-Playbooks.md](03-README-Runbooks-And-Playbooks.md)** — writing runbooks/playbooks that actually get used under pressure.
4. **[04-README-Postmortems-And-5-Whys.md](04-README-Postmortems-And-5-Whys.md)** — blameless postmortems and 5 Whys root cause analysis.

## Severity levels (P1–P4)

Severity is a **standardized, pre-agreed classification** applied the moment an incident is declared, so that response urgency, staffing, and communication cadence are decided by policy — not by whoever notices the problem first guessing how bad it feels. A typical P1–P4 scheme (naming/exact boundaries vary by org, but the shape is near-universal):

| Severity | Definition | Example | Response expectation |
|---|---|---|---|
| **P1 / SEV1** | Complete outage or critical business function down, affecting most/all users, no workaround | Payment processing down site-wide; total site outage; data loss in progress | Immediate page, all-hands, exec/stakeholder comms, war room, drop everything |
| **P2 / SEV2** | Significant degradation or a major feature broken, affecting a meaningful subset of users, workaround may exist | One region down; a core feature (search, checkout) badly degraded but not fully down | Immediate page, dedicated responder(s), comms to affected-team stakeholders |
| **P3 / SEV3** | Minor functionality impaired, small subset of users or a non-critical feature affected | A secondary feature broken; elevated error rate on a low-traffic endpoint | Business-hours response, ticket/queue, no page required |
| **P4 / SEV4** | Cosmetic or trivial issue, no meaningful user impact | Typo, minor UI glitch, non-blocking log noise | Backlog item, fixed in normal sprint work |

**The classification is a decision the incident commander (or whoever declares) makes explicitly, early, and revisits as new information arrives** — severity can be upgraded or downgraded mid-incident as scope becomes clearer (an incident that looked like a P3 "elevated errors on one endpoint" can be upgraded to P1 once you realize that endpoint is the payment gateway). What matters for interview purposes is being able to articulate *why* a specific scenario is a P1 vs P2 — the reasoning (blast radius, business criticality, workaround availability, data integrity risk), not memorized labels, is the signal an interviewer is looking for.

**Why severity must be pre-agreed policy, not improvised:** it drives concrete, automatic downstream decisions — who gets paged (file 2's escalation policy), whether a status page update goes out, whether execs get pinged, whether the postmortem is mandatory (usually mandatory for P1/P2, optional for P3/P4). If severity is subjective and argued about in the moment, you lose precious minutes exactly when speed matters most.

## On-call best practices

On-call is the operational backbone that makes "someone always owns actively-degrading production issues" possible outside business hours. Practices that separate sustainable on-call programs from ones that burn people out:

- **Bounded rotation length.** A common pattern is one week on, then several weeks off, rotating across a large enough pool of engineers that no one is on-call more than roughly 1 week in 4-6. Shorter/more frequent rotations reduce fatigue per shift but increase context-switching; the right balance depends on page volume.
- **A defined, humane response-time SLA per severity**, e.g., "P1 acknowledge within 5 minutes, P2 within 15 minutes" — not "as soon as possible," which is unmeasurable and unfair to hold anyone accountable to.
- **Primary + secondary on-call.** A secondary on-call exists specifically so that if the primary doesn't acknowledge within the SLA (asleep, phone dead, no signal), escalation has somewhere to go automatically — this is exactly what an escalation policy (file 2) encodes.
- **Compensation/comp-time for on-call burden**, especially for pages outside working hours — treating on-call as unpaid extra work is one of the most common, most corrosive on-call anti-patterns, and a major driver of attrition on infra/SRE teams.
- **Actionable pages only.** Every page that fires should require a human action *right now* — anything that can wait until morning should be a ticket, not a page (directly related to Day 96's SLO-based alerting philosophy: page on symptoms that threaten the error budget, not on every internal anomaly). A high volume of non-actionable pages is the single biggest predictor of on-call burnout and alert fatigue (people start ignoring pages, including the real ones).
- **Handoff discipline.** At the start/end of a rotation, the outgoing on-call should hand off known ongoing issues, recent changes, and anything flaky — an on-call engineer walking in blind to an already-degrading system wastes critical minutes re-discovering context the previous person already had.
- **Track and review page volume/quality over time.** If a service pages the same on-call engineer 15 times a week for the same root cause, that's a toil signal (Day 98) demanding a fix, not just "more tolerance."

## Points to Remember

- Severity levels (P1-P4 or SEV1-4) are a pre-agreed classification scheme that drives paging, staffing, and communication policy automatically — not a subjective, argued-in-the-moment judgment call.
- Severity can and should be revised as an incident's scope becomes clearer — don't anchor to the initial guess.
- Primary + secondary on-call with automatic escalation-on-non-ack is the standard defense against a single point of human failure (asleep, unreachable).
- Every page should be actionable *right now*; anything else belongs in a ticket queue, not a 3am phone buzz — this is the direct link between good alerting design (Day 96) and sustainable on-call.
- On-call sustainability (rotation length, compensation, actionable-only paging, handoff discipline) is itself an engineering responsibility, not just an HR/scheduling afterthought.

## Common Mistakes

- Treating severity as fixed at declaration time and never revisiting it as new facts emerge, leading to a P1 sitting mis-labeled as P3 (under-resourced) or a minor P3 issue getting P1-level executive attention it doesn't warrant.
- Paging on non-actionable, "just FYI" conditions — training the on-call engineer to develop alert fatigue, which is exactly how the *actually* critical page gets missed or delayed weeks later.
- Running on-call with no secondary/escalation path, so a single missed page (dead phone battery, no signal) means nobody responds until someone else notices independently.
- Not tracking recurring, repetitive pages as a toil signal — accepting "the same alert 10 times a week" as normal instead of treating it as a bug to fix (see Day 98).
- Treating on-call as free overtime with no compensation or comp-time — a fast path to attrition on the team that's supposed to be your most reliable line of defense during outages.
