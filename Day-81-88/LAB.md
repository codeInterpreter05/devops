# Day 81-88 — Lab: Phase 2 Consolidation & Interview Prep

**Goal:** Leave this 8-day block with two recorded mock interviews and honest feedback on them, a written system-design answer strong enough to reuse in a real interview, a portfolio with no obvious gaps a reviewer would flag, and an AWS SAA-C03 study schedule already running — in short, real evidence you're ready for Phase 3, not just a feeling that you are.

**Prerequisites:** Your Days 61-80 notes (CI/CD pipelines, GitOps, DevSecOps, supply chain security, deployment strategies), a way to record yourself (phone, Zoom/Meet recording, or OBS), access to your Phase 0-2 portfolio repos, and an AWS free-tier account for Day 3 file's kickoff work.

This lab is structured as a single 8-day arc, not four independent labs — do the parts in order, each builds on the last.

---

### Days 1-2 — Mock interview session 1 + gap-analysis pass (Days 61-70)

1. Run the **3-pass gap-analysis method** from [01-README-Phase2-Review-Strategy.md](01-README-Phase2-Review-Strategy.md) against the first half of Phase 2 (CI/CD pipelines, GitOps — Days 61-70 territory). Flag anything you can't explain smoothly in 60 seconds.
2. Re-study only the flagged items from your own source notes.
3. **Record mock interview session 1.** Structure it as a real 45-60 minute loop:
   - 10-15 minutes of behavioral/background questions (be ready to talk through a real CI/CD or GitOps problem you've solved, even a lab exercise, using a structured format like STAR — Situation, Task, Action, Result).
   - 30-40 minutes on a system design or deep-technical question pulled from your Days 61-70 material (e.g., "walk me through what happens when a developer merges a PR, all the way to production" or "how would you add a canary deployment to an existing pipeline").
   - Do this with a peer, a DevOps Discord community member, or — if no one is available — record yourself answering out loud, unscripted, from a written prompt you didn't write the answer to in advance.
4. **Self-grade or get peer-graded** using the rubric in [CHEATSHEET.md](CHEATSHEET.md) immediately after, while it's fresh. Write down the two or three biggest weaknesses in a running notes file — you'll revisit this list on Days 7-8.

**Success criteria:** You have a recording (or peer notes) of a full mock interview covering CI/CD + GitOps, a completed rubric score, and a written list of specific weaknesses (not "I did okay" — actual gaps like "I couldn't clearly explain drift detection" or "I rambled on the behavioral question about a production incident").

---

### Days 3-4 — System design doc + gap-analysis pass (Days 71-80)

1. Continue the gap-analysis pass, now against the second half of Phase 2 (DevSecOps, supply chain security, deployment strategies — Days 71-80 territory).
2. **Write a full system design document** for "Design a CI/CD platform for 500 engineers" — not bullet notes, an actual structured doc you could hand to someone. Use the 8-step worked answer in [02-README-System-Design-CICD-Platform.md](02-README-System-Design-CICD-Platform.md) as your outline, but write it in your own words with your own diagram (ASCII is fine). Sections to include at minimum:
   - Clarifying assumptions you're making
   - High-level architecture diagram
   - Self-service/scaling story for 500 engineers specifically
   - Security gates, in order, with rationale for the ordering
   - GitOps promotion flow across environments
   - Observability/rollback approach
   - At least 3 explicitly stated tradeoffs
3. Read your doc out loud, end to end, timing yourself — target 12-15 minutes for a full unprompted walkthrough (roughly what you'd get before an interviewer starts interjecting).

**Success criteria:** A written design doc you could actually hand to a hiring manager, that you can present out loud in 12-15 minutes without reading it verbatim, and that explicitly states at least 3 tradeoffs unprompted.

---

### Days 5-6 — Portfolio audit and gap-fixing

1. Go through every portfolio repo/project from Phase 0, Phase 1, and Phase 2. For each, check:
   - Does the README explain *what it does and why*, not just *how to run it*? A reviewer skimming for 90 seconds should understand the point of the project.
   - Is there evidence of the specific skill it's meant to demonstrate (e.g., a "CI/CD project" that doesn't actually show a working pipeline config is a gap)?
   - Are there dead links, TODOs left in, broken build steps, or secrets accidentally committed? (Run a secret scanner over your own repos if you haven't — practicing what Days 71-80 preached.)
   - Does it show your *most recent* skill level, or is it stale enough that it undersells where you actually are now?
2. Make a prioritized list: which 2-3 gaps would most hurt you if a reviewer found them first. Fix those first — you will not have time to perfect every repo, and a partial fix on the highest-impact gap beats a full fix on a low-impact one.
3. Fix the top gaps. This might mean: rewriting a weak README, adding a missing pipeline config, removing a stale/broken project, or adding a short "what I'd do differently" note to an old project (this alone signals growth to a reviewer, cheaply).

**Success criteria:** A written before/after list of the top 2-3 portfolio gaps and what you changed, and every remaining public repo has a README that explains its purpose clearly enough that a stranger understands it in under 2 minutes.

---

### Days 7-8 — Mock interview session 2 + AWS SAA-C03 schedule + Phase 3 readiness check

1. Set up your **AWS SAA-C03 study schedule** using the plan in [03-README-AWS-SAA-C03-Kickoff.md](03-README-AWS-SAA-C03-Kickoff.md): a fixed daily time slot, the 3-phase approach (skim domains → hands-on labs → baseline practice exam), and a target date for a first full practice exam.
2. **Record mock interview session 2.** This time, deliberately target the weaknesses you wrote down after session 1 (Days 1-2) — if you froze on GitOps drift detection last time, make sure this session's questions touch it again. Also include the flagship question verbatim: *"Design a CI/CD platform for 500 engineers"* — deliver your Days 3-4 written doc from memory, out loud, without reading it.
3. Grade session 2 with the same rubric. Compare scores against session 1 — you're looking for measurable improvement on the specific weaknesses you targeted, not just a vague "felt better."
4. Run through the **Phase 3 readiness checklist** below. Be honest — this is the actual gate for moving on.

**Success criteria:** Session 2's rubric score is higher than session 1's, specifically on the weak areas you targeted; your SAA-C03 study schedule has a concrete daily slot and a dated first practice-exam target; and you can check off the readiness checklist below without hand-waving any item.

---

## Final checklist — how do you know you're ready for Phase 3?

Go through this honestly. If more than one or two items are "no," spend an extra day on that specific gap before moving on — it's cheaper now than mid-way through Phase 3.

- [ ] I can explain CI/CD pipeline design (stages, artifact promotion, caching/parallelization) out loud, unprompted, in under 2 minutes.
- [ ] I can explain GitOps (reconciliation loop, push vs. pull, drift detection) out loud, unprompted, in under 2 minutes.
- [ ] I can explain shift-left DevSecOps (SAST vs. SCA vs. DAST, policy as code) and give a concrete example of each.
- [ ] I can explain supply chain security basics (SBOM, image signing, SLSA provenance, dependency pinning) without confusing the terms.
- [ ] I can compare rolling/blue-green/canary deployments and state what each optimizes for and what it costs.
- [ ] I have delivered the full "CI/CD platform for 500 engineers" answer out loud, from memory, in 12-15 minutes, including at least 3 unprompted tradeoffs.
- [ ] I have two recorded mock interviews, graded with the rubric, and can point to a specific measured improvement between them.
- [ ] Every public portfolio repo has a clear README and no embarrassing gaps a reviewer would notice in the first 90 seconds.
- [ ] I have a running AWS SAA-C03 study schedule with a protected daily time slot and a dated first practice-exam target.
- [ ] I have a written, prioritized list of my remaining weak spots (not "none" — everyone has some; the point is knowing what they are).

If you can check every box, you're ready to start Phase 3. If not, that's exactly what this block was for — better to find the gap here than in a real interview loop.
