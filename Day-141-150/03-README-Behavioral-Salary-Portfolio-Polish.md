# Day 141-150 — Final Interview Blitz III: Behavioral Prep, Salary Negotiation, and Portfolio Polish

**Phase:** 4 – Advanced/Specialization | **Week:** W25 | **Domain:** Interview Prep | **Flag:** 📌

## Brief

Technical depth gets you to the offer stage; how you handle behavioral questions and negotiate the offer determines what that offer actually looks like. This file covers the three things that are easy to under-prepare because they don't feel as "technical" as the rest of this roadmap: structured behavioral answers (STAR), the fundamentals of negotiating comp once an offer exists, and the final polish pass on the public artifacts (GitHub, LinkedIn) a hiring manager will actually look at before they ever schedule a call with you.

## STAR — structure, not script

**STAR** = **S**ituation, **T**ask, **A**ction, **R**esult. It exists because unstructured behavioral answers drift — candidates either ramble through context forever and never get to what they actually did, or jump straight to "and it worked out great" with no concrete detail an interviewer can probe. STAR forces a shape:

- **Situation** — brief context. Where, when, what was the setup. 2-3 sentences, not a full backstory.
- **Task** — what specifically was *your* responsibility or goal in that situation (not the team's — yours).
- **Action** — what *you* actually did, step by step. This is the longest section and the one interviewers listen to most closely — vague action ("I worked with the team to fix it") is the single most common way a STAR answer fails.
- **Result** — what happened, ideally quantified (time saved, incidents prevented, percentage improvement), plus what you'd do differently if relevant. A result with no number isn't wrong, but a number makes it verifiable and memorable.

### Worked example

**Prompt:** "Tell me about a time you had to make a decision with incomplete information."

> **Situation:** During a production incident, `checkout-service`'s error rate spiked to 8% around 2am. Two on-call engineers were paged; I was primary.
>
> **Task:** I needed to decide within minutes whether to roll back the deploy that had gone out four hours earlier or investigate further, without full certainty the deploy was the actual cause — the timing was suspicious but not conclusive from the dashboards alone.
>
> **Action:** I checked the deploy timeline against the error spike's start time first (a five-minute check, not a guess) and saw the correlation was weak — the spike started nearly four hours after the deploy, not right after it, which argued against a straightforward bad-deploy rollback. I widened the search instead of committing to a rollback: pulled up the trace for a sample of failing requests, found they all touched the same downstream dependency (a payment gateway), and confirmed via its status page that the *gateway* was degraded, not our service. I made the call to *not* roll back — a rollback would have cost real deploy time and changed nothing, since the root cause was external — and instead added a circuit breaker fallback and posted a status update saying we were tracking a third-party degradation.
>
> **Result:** The gateway recovered on its own about 20 minutes later; our error rate returned to baseline immediately once it did. Afterward, I wrote a runbook entry specifically for "check deploy-timing correlation before rolling back" so the next on-call engineer wouldn't default to rolling back on pattern-match alone. That runbook has been used at least twice since without me being paged.

Notice what this example does: it names a real, specific decision (roll back or not), shows the actual reasoning (not "I used my judgment" — the concrete steps: timing check, trace sample, external status page), and closes with both a result and a durable change (the runbook), which signals the kind of "I fixed the system, not just the incident" thinking senior interviewers listen for.

### Build a bank, not a script

Prepare 6-8 stories in advance, each mapped to the behavioral categories interviewers reliably draw from, and know which prompts each story can answer (a good story usually covers 2-3 categories):

- A mistake you made and what changed afterward
- A conflict or disagreement with a teammate/another team
- A time you had to make a decision with incomplete information (as above)
- A time you influenced something without formal authority
- A time you had to prioritize under real time/resource pressure
- A time you received tough feedback

Do **not** memorize word-for-word scripts — memorized delivery sounds memorized, and a good interviewer's follow-up question ("what would you do differently?") exposes a script instantly because you haven't actually thought past the ending you rehearsed. Know the shape and the key facts; let the exact wording vary each time you tell it.

## Salary negotiation fundamentals

Negotiation is a skill, not a personality trait — most of the leverage comes from preparation done *before* the conversation, not cleverness during it.

- **Know your number before you're asked for one.** Research total compensation (not just base) for the role/level/location using levels.fyi and similar sources. Total comp = base + bonus + equity (annualized) + benefits — comparing only base salary between offers is a common, costly mistake.
- **Avoid giving the first number if you can.** "What are your salary expectations?" early in a process is often better answered with "I'd like to learn more about the role and scope first, but I'm sure we can find a number that works if it's a mutual fit" — the first number anchors the whole negotiation, and you want the anchor set by market data, not by you guessing low out of nerves.
- **If forced to give a range, give a range, and anchor it high.** State a range where your *minimum* is a number you'd genuinely be satisfied with — companies frequently negotiate toward the bottom of a stated range, not the middle.
- **Negotiate the full package, not just base.** Sign-on bonus, equity refresh timing, start date, remote/relocation terms, and title are all negotiable levers, sometimes with more flexibility than base salary itself (base often has tighter internal banding than the other components).
- **A competing offer is real leverage — a bluffed one is not.** If you have a second offer, you can use it honestly ("I have a competing offer at $X total comp, but this role is my preference — can you get closer to that?"). Do not invent one; it's discoverable and destroys trust if caught, and recruiters compare notes more often than candidates expect.
- **Silence is a tool.** After you state a counter, stop talking. The urge to immediately soften your ask ("...but I'm flexible, of course") undercuts the ask itself before the other side has even responded.
- **Get the final offer in writing before you resign anywhere else** or make any commitment based on it — verbal numbers shift; a written offer letter is the actual agreement.
- **It's very rarely a one-shot, take-it-or-leave-it moment** — most companies expect at least one counter and have budgeted room for it; not negotiating at all is the single most common way candidates leave money on the table, more common than negotiating "too aggressively."

## GitHub and LinkedIn — final polish checklist

By this point in the roadmap you have real portfolio material (Phase 0-3 projects, the 5 system design docs from file 2, certifications if pursued). This pass is about making sure a reviewer sees it clearly, not about adding more content.

**GitHub:**
- Profile README exists and states, in the first two sentences, what you do and what you're looking for — a reviewer scans for ~90 seconds before deciding whether to look deeper.
- Pinned repos (use all 6 slots) show your *best*, most relevant work — not your oldest or most numerous. Swap out anything stale or half-finished.
- Every pinned repo's own README explains *what it does and why* in the first paragraph, not just installation steps — see the Day 81-88 portfolio-audit criteria if you haven't already run that pass.
- No leaked secrets, no embarrassing TODOs, no broken build/CI badges on anything pinned — run a secret scanner over your own repos if you haven't (practicing what Phase 2 preached).
- Contribution graph doesn't need to be solid green — a lumpy graph with clearly substantial projects reads better than daily commits of trivial changes made to game the graph.

**LinkedIn:**
- Headline states role + specialization, not just a job title copied from your current employer's internal naming (e.g., "DevOps/Platform Engineer — Kubernetes, CI/CD, Observability" beats "Senior Systems Engineer II").
- "About" section: 3-4 sentences, what you do, what you've shipped, what you're looking for next — written for a recruiter skimming, not a personal essay.
- Experience bullets are outcome-oriented and quantified where possible ("reduced deploy time from 25 to 8 minutes by parallelizing CI" beats "responsible for CI/CD pipelines").
- Skills section matches what you can actually defend in an interview — don't list a tool you touched once in a tutorial three phases ago if a follow-up question would expose shallow familiarity.
- Open-to-work / recruiter-visibility settings are deliberately set (on if actively looking, and configured to signal the right role type) — many candidates forget this is even a setting.
- At least one visible artifact linked (a portfolio site, a standout repo, a published design doc from file 2) — a profile that's all text with nothing to click through undersells real work.

## Points to Remember

- STAR is a structure for *your* thinking, not a script to recite — know the shape and facts of 6-8 stories, let wording vary naturally each time.
- The strongest behavioral answers name a specific action *you* took and a result you can quantify — "the team decided" and "it went well" are the two most common ways a STAR answer loses credibility.
- Negotiation leverage comes almost entirely from preparation before the call: know total comp market data, avoid anchoring low by going first, and negotiate the whole package, not just base.
- Silence after stating your ask is a deliberate, useful tool — resist the urge to immediately walk it back.
- Portfolio polish at this stage is about clarity and curation (pin your best work, cut anything stale or broken), not about adding volume.

## Common Mistakes

- Answering behavioral questions with team-level narration ("we decided," "the team fixed it") instead of naming your specific individual contribution — interviewers are assessing you, not your team.
- Giving a result with no outcome stated at all ("...and then it was resolved") — always close with what actually happened and, where possible, a number.
- Blurting out a salary number in the first call before doing any market research, out of a desire to seem cooperative or avoid an awkward pause.
- Accepting the first offer immediately without any counter, out of fear of seeming difficult — most offers have negotiation room built in, and not using it is the most common way candidates underpay themselves.
- Leaving stale, broken, or half-finished repos pinned on GitHub because "I'll fix it later" — a reviewer sees the current state, not your intentions.
- Treating the LinkedIn headline/About section as a one-time setup from years ago and never revisiting it to reflect where your skills actually are now.
