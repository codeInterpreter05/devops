# Day 76 — CI/CD & DevSecOps Review: Gaps Review & Documentation

**Phase:** 2 – CI/CD & Security | **Week:** W12 | **Domain:** Review

## Brief

Review days aren't filler — they're where scattered knowledge from 15+ days of Phase 2 (testing, SAST, container security, signing, GitOps, compliance, migrations) either consolidates into something you can *narrate under pressure* or quietly evaporates. The specific skill being trained today isn't "learn something new," it's **self-audit**: identifying what you actually understand deeply versus what you've only seen once, and closing that gap before it surfaces as a bad answer in an interview or a bad decision in production. This is also, not coincidentally, exactly what a senior engineer does at the end of any project — a retro/gap-review discipline, not a one-off study technique.

This day is split into two files:

1. **This file** — a structured method for reviewing Phase 2 gaps and updating documentation.
2. **[02-README-Pipeline-Audit-Optimisation.md](02-README-Pipeline-Audit-Optimisation.md)** — auditing a real pipeline for security issues and optimizing it for speed/cost.

## A structured gaps-review method (don't just "reread your notes")

Passive re-reading gives a false sense of mastery — you recognize the material, which feels like knowing it, but recognition and recall are different skills, and interviews test recall. Use active methods instead:

1. **Closed-book topic dump.** For each Phase 2 domain (CI pipeline design, SAST/DAST, container/image security, secrets management, signing/provenance, GitOps, compliance, DB migrations), write from memory — no notes — everything you can recall about it in 5 minutes. Compare against your actual notes afterward. The *gaps between what you wrote and what's in your notes* are your real gap list, not a guess.
2. **Explain-out-loud test.** Pick 3 random topics from the list above and explain each one out loud, unscripted, as if to an interviewer, for 90 seconds. Recording yourself (even just audio) and listening back is uncomfortable but extremely effective — you'll hear your own hedging ("I think it's something like...") which is a precise signal of a shaky topic.
3. **Interview-question cross-check.** Every day file in this repo has an "Interview question to prep" — go back through Days 61–75 and answer each cold, timed to under 2 minutes each (a real interview won't wait for you to compose a perfect answer).
4. **Build vs. describe gap check.** For each major tool (Trivy, cosign, ArgoCD, Prowler, Checkov, kube-bench, Flyway/Alembic), ask: *could I set this up from scratch right now, or do I only remember what it does when I read about it?* Tools you can only "describe" but not "build with" are prime candidates for a quick re-lab, not just re-reading.

## Documentation update: why this matters as much as the review itself

A gaps review that stays in your head is only useful until you forget it again. The concrete deliverable today is updating your own running notes/documentation so that future-you (in an interview prep session 3 months from now, or mid-incident at 2 a.m.) has a fast, accurate reference — not because "documentation" is a box to check, but because the same discipline you're expected to apply at a real job (runbooks, ADRs, postmortems) starts with this habit.

Concretely, for each identified gap:

- **Add a short, concrete example** to your notes for anything you could only explain abstractly — a real command, a real YAML snippet, a real failure you caused and fixed. Abstract explanations are the first thing that fades from memory; a specific example ("the time Trivy blocked a build because of an old `node:14` base image") sticks.
- **Cross-link related days.** Phase 2 topics are deeply interconnected (Day 73's pipeline literally uses Day 74's Checkov and Day 75's migration-gating pattern) — if your notes for one day don't reference the others, add the links now while the connections are fresh.
- **Flag anything you still don't trust yourself on** explicitly (a `## Still shaky` section, or similar) rather than silently hoping it'll be fine — a flagged gap is a plan; a silent gap is a surprise waiting to happen in an interview.

## Points to Remember

- Recognition (rereading notes and nodding along) is not recall (producing the answer from nothing) — interviews test recall, so your review method must too.
- Closed-book dumps, out-loud explanation, and timed interview-question answers surface real gaps; passive rereading mostly doesn't.
- Distinguish "can describe" from "can build" per tool — the second is a much stronger and more interview-resilient signal of understanding.
- Documentation updates aren't busywork — the same habit (runbooks, ADRs, postmortems) is a core professional skill, and today is deliberate practice for it.
- A `## Still shaky` list is more valuable than silence — an explicitly flagged gap is something you can plan to close; an unflagged one just waits to be discovered at a bad time.

## Common Mistakes

- Treating "review day" as "reread everything once, quickly" — this produces recognition, not recall, and gives false confidence heading into an interview.
- Reviewing every topic with equal depth regardless of actual gap size — spend disproportionate time on topics your closed-book dump revealed as weak, not on topics you already know cold.
- Updating notes with more copied reference material instead of your own worked examples and the specific mistakes you personally made in the labs — generic notes don't stick; your own concrete experience does.
- Skipping the "explain out loud" step because it feels awkward — this is precisely the step that reveals hedging/uncertainty that silent review never surfaces.
- Not revisiting the interview questions from earlier days at all, assuming "I read the answer once, that's enough" — cold, timed recall of exactly those questions is the most direct rehearsal for the actual interview format.
