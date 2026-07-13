# Day 141-150 — Final Interview Blitz II: The 5-Architecture System Design Portfolio

**Phase:** 4 – Advanced/Specialization | **Week:** W25 | **Domain:** Interview Prep | **Flag:** 📌

## Brief

A system design interview rewards candidates who've *already written the answer down once*, carefully, before they ever have to say it out loud under pressure. You've been building the raw material for this across the entire roadmap — the CI/CD platform design from Day 81-88, the full observability stack from Day 101-109, and everything Phase 4 covered about service mesh, platform engineering, and multi-cloud. This block's job is to take five of those and turn them into **polished, standalone, reusable documents** — the kind you could genuinely hand to an interviewer or a hiring manager, not bullet notes only you could follow.

## What makes a system design doc actually strong

Weak system design writing reads like a diagram with captions. Strong system design writing reads like a decision record — it shows *why* each choice was made, not just what was chosen. Every one of your five docs should have these sections, in this order:

```
1. PROBLEM STATEMENT & CLARIFYING ASSUMPTIONS
   - Restate the problem in your own words
   - List the assumptions you're making where the prompt is ambiguous
     (scale, team size, compliance, existing tooling) — state them even
     if no one asked, because a real interviewer usually won't hand you
     every number upfront either

2. HIGH-LEVEL ARCHITECTURE
   - One diagram (ASCII is fine), end to end
   - Name every major component and the direction data/control flows
     (push vs. pull, sync vs. async)

3. DEEP DIVE (pick 1-2 areas, go genuinely deep)
   - Don't spread deep-dive attention evenly across everything — a
     reviewer/interviewer wants to see real depth somewhere, not
     shallow coverage everywhere

4. SECURITY
   - Where are the gates/controls, in order, and why that order
   - AuthN/AuthZ, secrets, audit trail, defense in depth

5. SCALING / FAILURE MODES
   - What breaks first as load grows, and what you'd change
   - What happens when a key component is slow or fully down

6. TRADEOFFS (at least 3, stated unprompted)
   - Buy vs. build, centralized vs. federated ownership, cost vs.
     resilience, consistency vs. availability — whatever's genuinely
     in tension for this specific design
```

This is the same skeleton as the system-design-answer framework from Day 81-88 (see CHEATSHEET.md) — the difference here is you're writing the *complete, polished* version once, in advance, rather than improvising the structure live every time. Target 800-1500 words per doc, plus a diagram — long enough to show real depth, short enough that you (or a reviewer) can read it in under 10 minutes.

## The five architectures

You don't need to invent five designs from scratch — you need to **finish** five designs the roadmap already gave you the raw material for. Suggested set, chosen to span the full roadmap rather than cluster in one phase:

### 1. A CI/CD platform for a large engineering org

Source material: Day 81-88's full worked answer ("Design a CI/CD platform for 500 engineers"). Polish that into your own words, your own diagram, with the golden-path self-service story, ordered security gates, GitOps promotion flow, and at least three tradeoffs. This is the single most commonly asked infra system-design question — it should be your most rehearsed doc.

### 2. A multi-cluster GitOps setup

New synthesis, drawing on Phase 1 (Kubernetes architecture) and Phase 2 (GitOps/ArgoCD): design GitOps for a fleet of clusters (e.g., per-region prod clusters plus staging/dev), not just one. Cover a **hub-and-spoke** control-plane model (one management cluster running Argo CD, targeting N workload clusters via `ApplicationSet` generators or cluster registration), how you avoid config drift *between* clusters (shared base + per-cluster overlays — Kustomize or Helm values per environment), how a single bad config change is prevented from rolling out to every cluster simultaneously (progressive sync waves, canary clusters, or a staged rollout by region), and how you'd detect a cluster that's silently fallen out of reconciliation health.

### 3. A full observability stack

Source material: Day 101-109's Prometheus + Grafana + Loki + Tempo capstone. Polish this into the doc form — the correlation story (exemplars, derived fields, trace-to-logs) is what makes this design stand out over "here are three tools," so don't compress that section. Fold in the SLO/error-budget/burn-rate alerting design from the same block as the "what do you alert on" deep dive.

### 4. A multi-cloud strategy

New synthesis, drawing on Phase 4's multi-cloud/edge material plus AWS knowledge from Phases 1-3: design for a company that needs to run on two cloud providers (e.g., AWS + GCP), and be explicit about *why* — disaster-recovery independence from a single provider's regional outage, a specific customer/compliance requirement, cost leverage/negotiating power, or avoiding a single point of organizational risk. Cover the abstraction layer choice (Kubernetes as the portable compute layer, Terraform/Crossplane as the portable provisioning layer), what you deliberately do *not* try to abstract (managed databases, provider-specific networking primitives — abstracting everything usually costs more than it saves), data residency/replication strategy between clouds, and identity federation across two clouds' IAM systems. State the tradeoff explicitly: multi-cloud roughly doubles operational complexity and headcount need — justify it against a real driver, don't design it "because it sounds advanced."

### 5. A platform engineering internal developer platform (IDP)

New synthesis, drawing on Phase 4's platform-engineering material plus the self-service section from Day 81-88: design a Backstage-style IDP for an org where teams currently each hand-roll their own CI/CD, provisioning, and dashboards. Cover the **service catalog** (what a service's metadata/ownership record looks like and who maintains it), **golden-path templates** (scaffolding a new service — repo, pipeline, baseline dashboards, on-call rotation — in minutes, not a ticket), **the platform team's actual product surface** (what's self-service vs. what still requires a request), and how you'd measure whether the IDP is actually succeeding (a real metric, e.g., time from "I need a new service" to "first successful deploy," not just "we launched a portal").

## How to actually use this portfolio

- **Deliver each one from memory, out loud, in 10-15 minutes**, timing yourself, before you consider it done — a doc you can only read verbatim isn't ready, because a real interview will interrupt you and you need to be able to pick back up without losing the thread.
- **Publish the polished versions in your GitHub portfolio** (see file 3's portfolio checklist) — a design doc a reviewer can read in 10 minutes is one of the highest-signal things you can put in front of a hiring manager before you ever speak to them.
- **Use them as your system-design mock-interview material** (see file 1 and LAB.md) — sessions 4-5 in the mock-interview rotation are built specifically to deliver these live to a partner.
- **Update, don't archive, the two you already half-wrote** (CI/CD platform, observability stack) — you're not starting from zero on 2 of the 5, which is exactly why this 10-day block can fit five complete docs instead of two.

## Points to Remember

- A strong design doc is a decision record (why), not a diagram with captions (what) — the tradeoffs section is not optional decoration, it's the part that signals seniority.
- Reuse, don't duplicate, effort you already spent: docs 1 and 3 already have full worked answers from earlier in the roadmap — your job is polishing, not drafting from scratch.
- Deep-dive on 1-2 components per doc, not everything evenly — even coverage everywhere reads as recall, not judgment.
- Every doc needs at least 3 explicitly stated tradeoffs, unprompted — this is the single most consistent signal of a senior-level answer across every design doc reviewed in this roadmap.
- A design isn't "portfolio ready" until you can deliver it out loud, from memory, in the 10-15 minute range, and recover smoothly if interrupted mid-explanation.

## Common Mistakes

- Writing five designs that are all variations on "here's a Kubernetes architecture" — deliberately choosing topics that span different concerns (CI/CD, GitOps at fleet scale, observability, multi-cloud, platform engineering) is what makes the portfolio look broad instead of repetitive.
- Skipping the clarifying-assumptions section because "it's just for me, I know the assumptions" — writing them down is what trains you to actually state them out loud in a real interview instead of jumping straight to architecture.
- Over-polishing the diagram and under-investing in the tradeoffs section — a beautiful ASCII diagram with zero stated tradeoffs still reads as junior.
- Designing multi-cloud (or any "advanced" architecture) because it sounds impressive rather than stating a real driver for it — an interviewer who asks "why multi-cloud" and gets "for resilience" with no specifics will probe harder, not less.
- Never actually rehearsing delivery out loud — a doc that only exists as something you can read is not the same skill as being able to present it, and the gap between the two is exactly what a live interview exposes.
