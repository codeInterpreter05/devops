# Day 141-150 — Lab: The Final Interview Blitz

**Goal:** Leave this 10-day block — the last block of the entire 150-day roadmap — with 10 recorded mock interviews (at least 3 with real peers) and a tracked improvement trend, 5 polished system design documents you can each deliver from memory in 10-15 minutes, a bank of 6-8 STAR-format behavioral stories, salary-negotiation numbers researched and a script rehearsed, and a GitHub/LinkedIn presence that shows a stranger your actual current skill level in under 2 minutes.

**Prerequisites:** Every prior day's notes (you'll be drawing on all of them), a way to record yourself (phone, Zoom/Meet recording, OBS), at least one Pramp.com account set up, access to your full portfolio of repos from Phases 0-4, and a rough target comp range researched in advance (levels.fyi or equivalent) before Days 7-8.

This lab is a single 10-day arc, not four independent labs — do the parts roughly in order; each part assumes the previous one has already produced something to build on.

---

### Days 1-3 — Mock interviews 1-3 (peer-based) + gap analysis

Run the **3-pass gap-analysis method** (list from memory → explain out loud → targeted re-study, same method used in the Phase 2 consolidation block) against your entire roadmap once, at a high level, before session 1 — you're not re-studying everything, just triaging where your own confidence is lowest.

1. **Session 1 (Day 1):** Book a peer (Pramp or a Discord/community partner). Structure: 10 min behavioral, 30-40 min technical on **Phase 0-1 territory** (Linux, networking, Kubernetes, Terraform, AWS core), 10 min your questions. Record it.
2. **Session 2 (Day 2):** New peer if possible (different interviewer = different follow-up style = more realistic training signal). Technical focus: **Phase 2 territory** (CI/CD, GitOps, DevSecOps, supply chain). Record it.
3. **Session 3 (Day 3):** Technical focus: **Phase 3 territory** (observability, SLOs/error budgets, incident response). Record it.
4. After each session: grade it immediately using the rubric in [CHEATSHEET.md](CHEATSHEET.md), and write down the two or three biggest weaknesses in a running notes file — not "I did okay," specific gaps ("I couldn't clearly explain why GitOps prefers pull over push").

**Success criteria:** Three recorded, peer-graded mock interviews covering Phases 0-3 collectively, a completed rubric score for each, and a running list of specific named weaknesses you'll target later in this block.

---

### Days 4-6 — Write the 5 system design docs + mock interviews 4-5

1. Using the outline and the five suggested architectures in [02-README-System-Design-Portfolio.md](02-README-System-Design-Portfolio.md), write all five design docs across these three days:
   - **Day 4:** Polish the CI/CD platform doc (from Day 81-88's draft) and the observability stack doc (from Day 101-109's draft) into their final portfolio form — these two already have most of the thinking done, so treat this as editing, not drafting.
   - **Day 5:** Draft the multi-cluster GitOps doc and the multi-cloud strategy doc from scratch.
   - **Day 6:** Draft the platform-engineering IDP doc, then do a final consistency pass across all five (same section structure, each with at least 3 explicit tradeoffs, each 800-1500 words plus a diagram).
2. **Session 4 (Day 5 or 6):** Deliver one of your finished docs out loud, from memory, to a peer or self-recorded, timed to 10-15 minutes, then take live follow-up questions if it's a peer session.
3. **Session 5 (Day 6):** Deliver a second finished doc the same way, ideally the one you feel weakest presenting.

**Success criteria:** Five complete, written design docs meeting the structure in file 2, and two additional recorded sessions (bringing your total to 5) where you delivered a design doc from memory within the 10-15 minute target and could field at least one follow-up question without losing your structure.

---

### Days 7-8 — STAR behavioral prep + salary negotiation practice + mock interviews 6-7

1. Using [03-README-Behavioral-Salary-Portfolio-Polish.md](03-README-Behavioral-Salary-Portfolio-Polish.md), draft 6-8 STAR-format stories covering: a mistake and its fix, a conflict/disagreement, a decision made with incomplete information, influence without authority, prioritization under pressure, and tough feedback received. Write each as Situation/Task/Action/Result bullets — not full prose, just enough structure that you could speak it fluently without reading.
2. Research your target total-compensation range for the roles/levels/locations you're targeting (levels.fyi or equivalent) and write down: your target number, your walk-away minimum, and which package levers (sign-on, equity, start date) you'd prioritize if base is capped.
3. Run a **mock salary negotiation** with a peer playing the recruiter/hiring manager — practice not giving the first number, stating a range anchored high, and using silence after your counter. This doesn't need to be recorded, but do run it out loud with another person at least once; negotiating silently in your head is not the same skill.
4. **Session 6:** A behavioral-heavy mixed loop — have your partner pull from your story bank plus at least one unscripted "tell me about a time" prompt you haven't prepped for, to test whether your prepared stories flex to a slightly different question.
5. **Session 7:** Technical focus: **Phase 4 advanced territory** (service mesh/mTLS, platform engineering, FinOps, zero trust, CKS-adjacent security). Record it.

**Success criteria:** A written bank of 6-8 STAR stories you can deliver fluently without reading, a researched comp range with a stated walk-away number, at least one rehearsed negotiation roleplay, and two more recorded sessions (bringing your total to 7).

---

### Days 9-10 — Portfolio/LinkedIn polish + final mock interviews 8-10

1. Run the GitHub and LinkedIn checklists from file 3 end to end: pin your best 6 repos, verify every pinned repo's README explains *what and why* in its first paragraph, scan your own repos for leaked secrets, update your LinkedIn headline/About/experience bullets, and set your open-to-work visibility deliberately.
2. Link at least one of your five polished system design docs (file 2) somewhere visible in your portfolio (a repo, a pinned gist, a personal site) — a design doc a reviewer can read in 10 minutes before ever speaking to you is a strong differentiator.
3. **Sessions 8-10:** Run three full, mixed, end-to-end mock loops exactly like a real interview day — behavioral, then a technical deep-dive or system design pulled from anywhere in the roadmap, then your own questions. Deliberately re-target whatever weaknesses are still recurring on your running list from Days 1-3 and 7. Get at least one of these three back onto a real peer if your first three peer sessions used up your easiest-to-book partners — book ahead.
4. Compare your rubric scores across all 10 sessions. You're looking for a visible upward trend on the specific dimensions you targeted, not a perfect score on every session.
5. Run the **final readiness checklist** below, honestly.

**Success criteria:** Portfolio and LinkedIn polish complete against both checklists, all 10 mock interviews recorded and rubric-scored with a visible improvement trend on your previously flagged weaknesses, and the final readiness checklist below checked off without hand-waving any item.

---

## Final "ready for real interviews" checklist

This is the actual gate for the entire 150-day roadmap. Go through it honestly — if more than one or two items are "no," that's a specific, nameable gap, not a reason to panic; spend a focused extra session on exactly that gap before your first real interview.

**Phase 0 — Foundation**
- [ ] I can explain the Linux filesystem hierarchy, process management, and basic networking (TCP/IP, DNS) out loud, unprompted, in under 2 minutes each.
- [ ] I can walk through debugging a slow or failing service using real command-line tools, naming the actual commands, not just the concepts.

**Phase 1 — Core DevOps**
- [ ] I can explain Kubernetes architecture (control plane, kubelet, scheduler, reconciliation loops) and Terraform state/module patterns without confusing the two systems' models.
- [ ] I can explain core AWS services (VPC, IAM, EKS/ECS, RDS/DynamoDB) at the level SAA-C03 or a real architecture question would expect.

**Phase 2 — CI/CD & Security**
- [ ] I can explain CI/CD pipeline design (artifact promotion, caching/parallelization), GitOps (push vs. pull, drift detection), and shift-left security (SAST vs. SCA vs. DAST) cleanly and without conflating terms.
- [ ] I have delivered the "CI/CD platform" design doc from memory, unprompted, including at least 3 tradeoffs.

**Phase 3 — Observability**
- [ ] I can define SLI/SLO/SLA precisely and do the error-budget/burn-rate math live, not just recite the vocabulary.
- [ ] I can explain how metrics, logs, and traces correlate (exemplars, derived fields) and why that correlation — not just having three tools — is the actual point.

**Phase 4 — Advanced/Specialization**
- [ ] I can explain at least one Phase 4 advanced topic (service mesh/mTLS zero trust, platform engineering/IDP, FinOps, or multi-cloud strategy) with real depth, not just buzzwords.
- [ ] I have delivered at least 2 of my 5 system design docs from memory, live, and handled a follow-up question without losing structure.

**Interview mechanics**
- [ ] I have 10 recorded mock interviews, at least 3 with real peers, scored with a consistent rubric, showing a visible improvement trend on my own previously flagged weaknesses.
- [ ] I have 6-8 STAR-format behavioral stories I can deliver fluently without reading, covering mistakes, conflict, incomplete-information decisions, influence without authority, prioritization, and feedback.
- [ ] I have a researched target compensation range, a stated walk-away minimum, and have rehearsed a negotiation conversation out loud with another person at least once.
- [ ] My GitHub and LinkedIn both show a stranger my actual current skill level in under 2 minutes, with no stale/broken/embarrassing content on anything visible.
- [ ] I have a written, honest list of my remaining weak spots — not "none." Knowing exactly what they are, and having a plan for closing them, is itself part of being ready.

If you can check every box: you've completed the roadmap, and you're ready for real interviews. If not — that's exactly what finding it here, instead of in a real interview loop, was for.
