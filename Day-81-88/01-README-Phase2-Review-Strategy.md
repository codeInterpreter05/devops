# Day 81-88 — Phase 2 Consolidation I: Review & Gap-Analysis Strategy

**Phase:** 2 – CI/CD & Security | **Week:** W13-W14 | **Domain:** Review & Buffer | **Flag:** 📌

## Brief

This is not a new-content day — it's an 8-day consolidation block that closes out Phase 2 (Days 61-80: CI/CD pipelines, GitOps, DevSecOps, supply chain security, deployment strategies) before you move into Phase 3. The temptation at this point is to re-read your old notes cover-to-cover and call it "review." That's the weakest form of review there is — passive re-reading feels productive but barely touches long-term retention. What actually works, and what an interviewer's rapid-fire follow-up questions will expose if you skipped it, is **active recall**: closing the notes, explaining a concept out loud or on a whiteboard from memory, and only going back to source material for the specific things you got wrong or froze on.

This block is split into three files:

1. **This file** — the gap-analysis method and a topic-by-topic self-test matrix for everything in Days 61-80.
2. **[02-README-System-Design-CICD-Platform.md](02-README-System-Design-CICD-Platform.md)** — a full worked answer to the flagship system design question for this phase.
3. **[03-README-AWS-SAA-C03-Kickoff.md](03-README-AWS-SAA-C03-Kickoff.md)** — starting AWS SAA-C03 prep in parallel without derailing interview readiness.

The hands-on work (mock interviews, the design doc, portfolio fixes, the study schedule) lives in [LAB.md](LAB.md) — read this file first to understand the *method*, then go execute it there.

## Why a dedicated consolidation block, not just "more days"

Two well-established learning effects justify blocking out 8 full days for this instead of cramming it into a weekend:

- **The forgetting curve**: without deliberate review, recall of Day 61 material has decayed substantially by Day 80. A spaced pass right before the next phase re-consolidates it into longer-term memory — this is *the* mechanism, not a nice-to-have.
- **The testing effect**: the act of retrieving information (quizzing yourself, explaining out loud, answering a mock interviewer) strengthens memory far more than re-reading the same material an equivalent number of times. This is why the method below is built around self-testing, not re-reading.

Interviews are a retrieval-under-pressure exercise. If your only practice has been recognition ("oh yeah, I remember that from my notes"), you will freeze the first time someone asks a follow-up you didn't rehearse. This block exists to convert passive familiarity into active, speakable knowledge.

## The 3-pass gap-analysis method

Run this for each domain below, in order:

1. **Pass 1 — List from memory.** Without notes, write down every sub-topic you can recall for a domain (e.g., for "GitOps": declarative desired state, reconciliation loop, drift detection, push vs. pull deployment, Argo CD/Flux...). Time-box to 3 minutes per domain.
2. **Pass 2 — Explain out loud or on a whiteboard.** Pick each item from your list and explain it in 60 seconds as if to an interviewer — not just define it, but say *why it matters* and give a concrete example or failure mode. Anything you can't do smoothly is a **flag**.
3. **Pass 3 — Targeted re-study.** Go back to your source notes *only* for flagged items. Do not re-read entire days you already recalled cleanly — that's wasted time this late in the program.

This is deliberately front-loaded toward finding weaknesses fast rather than confirming what you already know.

## Self-test matrix — Days 61-80

Use this table as your Pass-1/Pass-2 checklist. For each row, the "cold-recall test" is the question a decent interviewer would actually ask; the "red flag" is what tells you the topic needs a re-study pass.

| Domain | Cold-recall test | Red flag if you can't answer |
|---|---|---|
| **CI/CD pipelines** | Draw the stages of a pipeline (build → test → package → scan → deploy) and explain what "pipeline as code" buys you over a GUI-configured job. | Can't explain artifact promotion (build once, promote the same artifact through envs) vs. rebuilding per environment — this is one of the most common CI/CD interview traps. |
| **CI/CD pipelines** | Explain caching and parallelization strategies (dependency caches, test sharding, matrix builds) and why they matter at scale. | Can't explain why a monolithic sequential pipeline becomes a bottleneck as team/repo size grows. |
| **GitOps** | Explain the reconciliation loop: an operator (Argo CD/Flux) continuously diffs live cluster state against a Git repo's declared state and corrects drift. | Can't distinguish **push** deployment (CI pushes changes to the cluster) from **pull/GitOps** deployment (an in-cluster agent pulls and applies) — and why pull is preferred for security (no cluster credentials leave the cluster). |
| **GitOps** | Explain drift detection and what happens when someone `kubectl edit`s a resource that's GitOps-managed. | Can't explain why Git becomes the single source of truth and audit trail for "what's running in production." |
| **DevSecOps** | Explain "shift-left" concretely: name where in the pipeline SAST, SCA, secret scanning, and DAST each run, and why order matters (fail fast on cheap checks first). | Can't distinguish SAST (static code analysis) from SCA (dependency/vulnerability scanning) from DAST (running-app testing) — conflating these is a fast interview red flag. |
| **DevSecOps** | Explain "policy as code" (OPA/Gatekeeper, Kyverno) and give an example of a policy you'd enforce at admission time in Kubernetes. | Can't explain the difference between a policy that *warns* vs. one that *blocks* (admission control) — and why you'd stage new policies in warn mode first. |
| **Supply chain security** | Explain what an SBOM is, what generates one, and what question it answers during an incident (e.g., a new CVE drops — which of our images contain the affected library?). | Can't explain image signing (cosign/Sigstore) and what problem it solves (verifying an image wasn't tampered with between build and deploy). |
| **Supply chain security** | Explain SLSA provenance at a high level — what it attests to (how/where/from-what-source an artifact was built). | Can't explain why pinning dependencies by digest (not just version tag) matters, or what a "dependency confusion" attack is. |
| **Deployment strategies** | Compare rolling, blue-green, and canary deployments: what each optimizes for, and the rollback cost/speed of each. | Can't explain what a canary strategy needs to actually work (traffic splitting, automated metric-based promotion/abort, not just "deploy to a few pods and hope"). |
| **Deployment strategies** | Explain feature flags as a *decoupling* mechanism — deploy code dark, then flip behavior on independently of a deploy. | Can't explain why blue-green needs double the infrastructure (or a fast cutover mechanism) and how that tradeoff differs from canary's incremental resource use. |

## Points to Remember

- Active recall (testing yourself) beats passive re-reading for retention — this is the single biggest lever you have in a review week, not more hours.
- Flag items during Pass 2 the moment you hesitate or ramble — don't let yourself "talk around" a gap and call it understood.
- Re-study only what's flagged. At this stage in the program, re-reading material you already recall cleanly is the highest-cost, lowest-value thing you can do with limited days.
- The self-test matrix above is a floor, not a ceiling — if your Day 61-80 notes covered something not listed here, add it to your own Pass-1 list.
- Portfolio gaps and mock interviews are just another form of retrieval practice under more realistic pressure — treat them with the same "flag it, then fix it" discipline (see LAB.md).

## Common Mistakes

- Re-reading all 20 days of notes linearly instead of self-testing first — this feels thorough but mostly reinforces what you already knew and burns the days you need for mock interviews and portfolio fixes.
- Skipping the "explain out loud" step and only doing it in your head — silently thinking "yeah I get it" is not the same skill as producing a coherent 60-second spoken answer under mild pressure, which is what interviews actually test.
- Treating every domain as equally weak and re-studying everything — the whole point of Pass 1/Pass 2 is to triage so you spend your limited days on actual gaps.
- Conflating "I recognize this term" with "I can explain this to an interviewer" — recognition is the weakest, most misleading signal of readiness.
- Doing the review in isolation and never testing it against another person (peer, mock interviewer, DevOps Discord) — self-assessment alone misses blind spots that only surface under real questioning.
