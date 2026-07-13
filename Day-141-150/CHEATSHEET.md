# Day 141-150 — Cheatsheet: Final Interview Blitz

## System-design-answer framework (recap — use for any design question)

```
1. REQUIREMENTS  (spend 5-10 min here — don't skip this)
   - Functional: what must the system actually do?
   - Non-functional: scale, latency, availability, compliance
   - Ask clarifying questions; state assumptions explicitly if unanswered

2. HIGH-LEVEL ARCHITECTURE
   - Draw the end-to-end flow, box by box
   - Name major components; note push vs. pull, sync vs. async

3. DEEP DIVE (pick 1-2 areas — depth beats even coverage)
   - Self-service/scaling story if org-scale is part of the question
   - Failure modes: what happens when this component is slow/down?

4. SECURITY
   - Where are the gates? (earliest/cheapest first)
   - AuthN/AuthZ, secrets handling, audit trail, defense in depth

5. TRADEOFFS  (say these UNPROMPTED — this is the senior-vs-junior tell)
   - Buy vs. build, centralized vs. federated ownership
   - Consistency vs. availability, cost vs. resilience
   - What would change at 10x the stated scale?
```

## Mock-interview self/peer grading rubric

Score 1-5 per dimension (1 = didn't do this, 5 = smooth and unprompted). Track this number across all 10 sessions — the goal is a visible trend, not a perfect score on any one session.

| Dimension | 1 (weak) | 3 (okay) | 5 (strong) |
|---|---|---|---|
| Requirements gathering | Jumped straight to solution | Asked a couple of clarifying questions | Systematically covered functional + non-functional first |
| Structure/communication | Rambled, hard to follow | Mostly linear | Clear signposting, easy to follow without notes |
| Depth on deep-dive | Vague, hand-wavy | Correct but surface-level | Concrete, named real mechanisms, explained *why* |
| Security awareness | Not mentioned unless asked | Mentioned generically | Specific gates, ordered, tied to real threats |
| Tradeoffs stated | None offered | Only when asked | 2-3+ stated unprompted, with reasoning |
| Time management | Ran out of time / rushed | Roughly on pace | Full time used well, room for follow-ups |
| Behavioral/STAR quality | No clear structure | Structure present, thin on result | Clear S-T-A-R, quantified impact |
| Composure under follow-up | Froze/defensive | Recovered but rattled | Treated follow-ups as normal, adjusted smoothly |

**Overall = average of the 8.** 4+ is strong; 2 or below is a flagged weakness for your next session.

## STAR template + filled example

```
SITUATION  — 2-3 sentences of context (where/when/what)
TASK       — what specifically YOU were responsible for
ACTION     — what YOU actually did, step by step (longest section)
RESULT     — what happened, quantified if possible + what you'd
             change / what durable improvement came out of it
```

**Filled example** ("Tell me about a decision made with incomplete information"):
```
S: 2am incident, checkout-service error rate spiked to 8%. I was primary on-call.
T: Decide fast whether to roll back a 4-hour-old deploy, without full certainty
   it was the cause.
A: Checked deploy-timing correlation first (weak — spike started ~4h post-deploy,
   not right after). Pulled traces for failing requests -> all touched the same
   downstream payment gateway. Confirmed via the gateway's own status page it was
   degraded externally. Decided NOT to roll back; added a circuit-breaker fallback
   and posted a status update instead.
R: Gateway recovered ~20 min later, error rate returned to baseline immediately.
   Wrote a runbook entry: "check deploy-timing correlation before rolling back."
   That runbook has prevented at least 2 unnecessary rollbacks since.
```

**Behavioral story bank to prep (6-8 stories, map each to 2-3 of these):**
```
- A mistake you made and what changed afterward
- A conflict/disagreement with a teammate or another team
- A decision made with incomplete information
- A time you influenced something without formal authority
- A time you had to prioritize under real time/resource pressure
- A time you received tough feedback
```

## Salary negotiation script/checklist

```
BEFORE THE CALL
[ ] Research total comp (base + bonus + equity annualized + benefits) for
    role/level/location — levels.fyi or equivalent
[ ] Decide: target number, walk-away minimum, priority order of levers
    (base > sign-on > equity > start date > title — reorder to your actual priority)

DURING THE CALL
[ ] If asked for expectations early, defer: "I'd like to learn more about
    scope first, but I'm confident we can find a number that works."
[ ] If forced to give a range, anchor HIGH — your stated minimum should be
    a number you'd genuinely be satisfied with.
[ ] State your counter, then STOP TALKING. Do not immediately soften it.
[ ] If you have a real competing offer, use it honestly and specifically
    ("$X total comp") — never bluff one.
[ ] Negotiate the full package, not just base — sign-on, equity refresh
    timing, start date, and title all have flexibility, sometimes more
    than base.

AFTER THE CALL
[ ] Get the final offer in writing before resigning anywhere else or
    making any commitment based on it.
[ ] Almost every offer has room for at least one counter — not
    countering at all is the most common way candidates leave money on
    the table.
```

## Portfolio / LinkedIn optimization checklist

```
GITHUB
[ ] Profile README states what you do + what you're looking for, in the
    first 2 sentences
[ ] All 6 pinned-repo slots used, showing your BEST and most RECENT work
[ ] Every pinned repo's README explains what/why in its first paragraph
[ ] No leaked secrets, dead links, TODOs, or broken CI badges on anything
    pinned — run a secret scanner over your own repos
[ ] At least one of your 5 system design docs linked/visible somewhere

LINKEDIN
[ ] Headline = role + specialization, not a copy-pasted internal title
[ ] About section: 3-4 sentences — what you do, what you've shipped,
    what's next — written for a skimming recruiter
[ ] Experience bullets are outcome-oriented and quantified where possible
[ ] Skills list matches what you can actually defend under a follow-up
    question
[ ] Open-to-work / recruiter visibility deliberately configured
[ ] At least one clickable artifact linked (portfolio site, standout
    repo, a published design doc)
```
