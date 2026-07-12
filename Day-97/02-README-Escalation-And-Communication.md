# Day 97 — Incident Management: Escalation Policies & Communication During Incidents

**Phase:** 3 – Observability | **Week:** W16 | **Domain:** SRE | **Flag:** ⚡ Interview-critical

## Brief

An escalation policy is the automated safety net that guarantees a page never silently disappears into an unresponsive phone. Communication discipline is the practice that keeps a P1 from turning into chaos — stakeholders demanding updates in five different Slack threads while the actual responders try to fix the problem. Both are mechanical, learnable skills, and both are frequently the actual differentiator between "a bad 20-minute blip" and "a bad 3-hour outage with a angry status page and a dozen confused customers," independent of how hard the underlying technical fix was.

## Escalation policies (PagerDuty-style)

An escalation policy defines, in advance, **who gets paged, in what order, and how long to wait before escalating further** if there's no acknowledgment. This is configured declaratively in tools like PagerDuty, Opsgenie, or Incident.io, and typically looks like:

```
Escalation Policy: "Checkout Service - P1/P2"
  Step 1: Page primary on-call (checkout-team rotation)
          -> wait 5 minutes for acknowledgment
  Step 2: If unacknowledged, page secondary on-call (checkout-team rotation)
          -> wait 5 minutes for acknowledgment
  Step 3: If still unacknowledged, page the team's Engineering Manager
          -> wait 10 minutes
  Step 4: If still unacknowledged, page the Director / declare a wider incident
          -> notify incident-response channel org-wide
```

Key mechanics worth knowing cold:

- **Acknowledge vs. resolve** are different actions — acknowledging tells the system "a human has seen this and is working it" (stops further escalation), while resolving marks the underlying issue fixed. Failing to acknowledge promptly (even if you're already looking at it, just via a different channel) triggers escalation to the next tier — a common false alarm that trains people to always ack quickly through the tool itself, not just informally in Slack.
- **Notification channels stack**: push notification → SMS → phone call, typically escalating in urgency/intrusiveness if the earlier channel goes unanswered within a short window (e.g., push for 2 minutes, then a phone call that keeps ringing until answered) — because push notifications alone are too easy to sleep through.
- **Scheduled overrides and maintenance windows** — the ability to say "I'm unavailable Tuesday 2-4pm, route around me" or "we're intentionally taking this system down for maintenance, suppress paging" — without this, planned changes generate false-positive pages that erode trust in the alerting system.
- **Service-to-team mapping** — escalation policies are usually attached to a *service* (e.g., "checkout-api"), and each service maps to the team/rotation that owns it. Getting this mapping wrong (or leaving new services unmapped) means an incident on a brand-new service pages nobody, or pages the wrong team, at the worst possible time.

**Why automated escalation matters over "just call someone":** during an actual P1 at 3am, manually figuring out who's on-call, finding their phone number, and calling them wastes minutes that directly extend the outage. Automated, pre-configured escalation removes that lookup-and-decide step entirely and guarantees a bounded worst-case time-to-human-response even in the single-point-of-failure case (primary on-call unreachable).

## Communication during incidents

Once an incident is being actively worked, communication has to serve two very different audiences simultaneously without letting either interfere with the other:

1. **The responders actually fixing it** — need a focused, low-noise channel (a dedicated incident Slack/Teams channel, a war room bridge) to coordinate technical actions without being interrupted by stakeholders asking "any update?" every three minutes.
2. **Stakeholders/customers who need status** — need a separate, regular cadence of plain-language updates (via a status page, a stakeholder-facing channel, or scheduled updates from the incident commander), so they're not tempted to interrupt the responders directly for information.

Practices that make this work:

- **A named Incident Commander (IC) role**, distinct from whoever is actually debugging. The IC's job is coordination and communication — tracking status, deciding severity, deciding when to escalate/involve more people, sending stakeholder updates — explicitly *not* hands-on-keyboard debugging. This separation exists because the same person can't simultaneously do deep technical work *and* keep a dozen stakeholders informed without one of those jobs suffering; splitting the roles lets the responder(s) focus and the IC absorb all the interruption load.
- **A single source of truth for incident status** — usually a running timeline in the incident channel or a tool like Incident.io/StatusPage, timestamped, so that anyone joining mid-incident (a newly-escalated engineer, a manager) can read the history instead of asking "what's happened so far" out loud and stealing attention.
- **Regular update cadence, even when there's "nothing new."** "Still investigating, next update in 15 minutes" is a valid, important update — silence during an incident is what causes stakeholders to start pinging responders directly, which is the exact interruption discipline is trying to prevent.
- **A public status page (e.g., StatusPage/Statuspage.io) for customer-facing incidents** — externalizes the "is it just me" question so it doesn't flood support channels, and gives customers a self-serve way to check status without contacting support, which reduces load on both support and engineering during the incident.
- **Plain language, no jargon, in customer/stakeholder-facing updates** — "we're investigating elevated error rates affecting checkout" instead of "p99 latency on the payment-gateway sidecar spiked due to connection pool exhaustion" — the audience for status-page updates isn't engineers.

## Points to Remember

- Escalation policies encode who gets paged, in what order, and how long to wait, entirely in advance — removing the "figure out who's on-call" lookup step from the critical path during a live incident.
- Acknowledge (stops escalation) and resolve (marks fixed) are distinct actions — always acknowledge promptly through the tool itself, even if you're already aware informally.
- The Incident Commander role is explicitly separate from the hands-on-keyboard responder — coordination/communication and deep technical debugging are two different jobs that shouldn't be done by the same person simultaneously during a serious incident.
- Maintain two separate communication channels: a low-noise responder channel for coordination, and a regular-cadence stakeholder/status-page channel for updates — mixing them causes both groups to interfere with each other.
- "Still investigating, next update in N minutes" is a legitimate, necessary update — silence is what drives stakeholders to interrupt responders directly.

## Common Mistakes

- Not acknowledging a page through the actual tool (just reacting in Slack informally) — the escalation timer keeps running and pages the next tier unnecessarily, sometimes waking up a whole additional layer of people for no reason.
- Letting the same person be both Incident Commander and the primary hands-on debugger during a serious, multi-hour P1 — one of the two jobs (usually communication) gets dropped, and stakeholders end up interrupting the debugger directly for status.
- Going silent for long stretches during an active incident because "there's nothing new to say yet" — this is exactly when stakeholders start pinging responders directly, fragmenting focus at the worst time.
- Leaving a newly-launched service without an escalation policy/on-call mapping configured, so its first real incident pages nobody (or pages a team with no context on it).
- Using deep technical jargon in customer-facing status page updates, leaving customers more confused (and more likely to open support tickets) than a plain-language summary would have.
