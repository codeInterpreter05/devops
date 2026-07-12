# Day 97 — Cheatsheet: Incident Management

## Severity levels

```
P1/SEV1  Complete outage / critical function down, most/all users, no workaround
         -> Immediate page, all-hands, exec comms, war room
P2/SEV2  Major degradation, meaningful subset of users, workaround may exist
         -> Immediate page, dedicated responder(s)
P3/SEV3  Minor functionality impaired, small subset affected
         -> Business-hours response, ticket, no page
P4/SEV4  Cosmetic/trivial, no meaningful user impact
         -> Backlog item

Severity can be revised up/down as scope becomes clear. Pre-agreed policy, not improvised.
```

## On-call best practices

```
- Bounded rotation (e.g. 1 week on / 4-6 weeks off)
- Defined ack SLA per severity (e.g. P1: ack <5min, P2: ack <15min)
- Primary + secondary, auto-escalation on non-ack
- Comp-time / pay for on-call burden
- Every page must be actionable RIGHT NOW or it shouldn't page (should be a ticket)
- Handoff notes at rotation start/end
```

## Escalation policy shape (PagerDuty/Opsgenie style)

```
Step 1: page primary          -> wait 5 min
Step 2: page secondary        -> wait 5 min
Step 3: page EM               -> wait 10 min
Step 4: page Director / widen incident

ack   = "a human is on it"  -> stops escalation
resolve = "issue is fixed"  -> different action, don't confuse the two
```

## Roles during an incident

```
Incident Commander (IC) -> coordination, comms, severity calls, escalation decisions
                            NOT hands-on-keyboard debugging
Responder(s)             -> hands-on-keyboard technical fix
Scribe (optional)        -> maintains the timestamped timeline live
```

## Communication cadence

```
- Dedicated low-noise responder channel (coordination only)
- Separate stakeholder/status-page channel (regular updates)
- "Still investigating, next update in 15 min" IS a valid update
- Status page: plain language, no jargon
```

## Runbook checklist

```
[ ] Clear trigger condition at the top
[ ] Exact, copy-pasteable commands (not descriptions)
[ ] Diagnosis steps BEFORE remediation, with expected output at each step
[ ] Branching paths for different root causes
[ ] Explicit escalation/rollback path if steps don't work
[ ] Linked directly from the alert
[ ] Updated after every incident that reveals a gap
```

## Postmortem template

```markdown
# Postmortem: <title>
## Summary        - what happened, impact, duration, severity
## Timeline        - timestamped, from logs/chat NOT memory
## Root cause(s)   - via 5 Whys / contributing-factors analysis
## Impact          - users/requests affected, error-budget consumed
## What went well / what went poorly
## Action items    - owner + due date + specific, trackable
```

## 5 Whys

```
Why #1: symptom            -> proximate technical cause
Why #2-4: dig deeper       -> keep redirecting "a person did X" into
                               "why did the system make X easy/uncaught"
Why #5: systemic cause     -> actionable, process-level fix

Caveat: real incidents are often multi-causal - supplement with a
        contributing-factors / fishbone analysis, don't force one linear chain.
```

## Blameless framing — the reframe

```
Blame-framed:      "Why did the engineer forget to close the connection?"
System-framed:     "Why did our system make that mistake easy to make
                     and hard to catch before production?"
```
