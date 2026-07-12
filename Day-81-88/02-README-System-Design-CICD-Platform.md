# Day 81-88 — Phase 2 Consolidation II: Mock Interview Playbook — "Design a CI/CD Platform for 500 Engineers"

**Phase:** 2 – CI/CD & Security | **Week:** W13-W14 | **Domain:** Review & Buffer | **Flag:** 📌

## Brief

This is the flagship system design question for Phase 2, and it's a *platform* design question, not a "draw a pipeline" question — the difference matters enormously. At 500 engineers, the hard problems aren't "how do I run `npm test` in CI," they're organizational: self-service so a platform team of 5-10 people doesn't become a bottleneck for 500 engineers, security gates that don't grind velocity to a halt, and a promotion flow that scales across many teams without becoming a free-for-all. This file walks through a complete worked answer, structured the way you should actually deliver it in an interview: clarify, then architecture, then how it scales to the org, then security, then tradeoffs.

## The system design interview format, briefly

A 45-60 minute system design round almost always rewards the same behavior: **spend the first 5-10 minutes on requirements before drawing anything.** Interviewers are explicitly watching for candidates who jump straight to "we'll use Jenkins and Argo CD" without first asking what's actually being optimized for. Structure your time roughly as:

1. Clarify requirements (functional + non-functional) — 5-10 min
2. High-level architecture — 10-15 min
3. Deep-dive into 1-2 areas the interviewer steers you toward (self-service, security, scaling) — 15-20 min
4. Tradeoffs and what you'd change at 10x scale — 5-10 min

## Step 1 — Clarifying questions to ask

For "design a CI/CD platform for 500 engineers," ask before designing:

- How many teams/repos? Monorepo, polyrepo, or a mix?
- What languages/stacks? (Affects build tooling diversity and caching strategy.)
- Deployment target — Kubernetes, VMs, serverless, mixed?
- Compliance requirements — regulated industry (finance/healthcare) needing audit trails and mandatory approvals, or a startup that can move faster?
- Existing tooling to build on, or greenfield?
- What's the current pain point driving this project? (Slow builds? Security incidents? Too many one-off pipelines?)

Stating your assumptions explicitly when the interviewer doesn't answer (e.g., "I'll assume polyrepo, Kubernetes-based, no heavy regulatory constraint, and design for that — flag me if that's wrong") shows judgment even without a real answer.

## Step 2 — High-level architecture

The backbone, end to end:

```
Developer commits → Source Control (GitHub/GitLab)
        │
        ▼
CI Orchestrator (build, test, package, scan)  — e.g. GitHub Actions / Buildkite / Jenkins / GitLab CI
        │
        ▼
Artifact Registry (immutable, versioned) — container images + libraries/packages
        │
        ▼
Security Gates (SAST, SCA, secret scan, image signing) — gate the artifact, not just the code
        │
        ▼
GitOps Config Repo (desired state per environment: dev → staging → prod)
        │
        ▼
CD Operator (Argo CD/Flux) — pulls and reconciles cluster state
        │
        ▼
Runtime environments (dev / staging / prod, per-team or per-service)
        │
        ▼
Observability + automated rollback loop
```

Key architectural decision to call out explicitly: **build the artifact once, promote the same immutable artifact through every environment.** Never rebuild per environment — that reintroduces "works in staging, breaks in prod" risk from environment drift in the build itself, and breaks the provenance chain you need for supply chain security (see Day 81-88's review file for SBOM/SLSA).

## Step 3 — Self-service layer (this is what makes it scale to 500 engineers)

At small scale, a platform team can hand-hold every pipeline. At 500 engineers, that team becomes the bottleneck unless most day-to-day work is self-service:

- **Golden-path pipeline templates**: a small number of reusable, versioned pipeline definitions (e.g., "standard Node service," "standard Python service," "standard Helm-deployed service") that teams inherit from rather than write from scratch. Teams override only what's genuinely different about their service.
- **Internal developer platform / catalog** (Backstage-style): a self-service portal where a team can scaffold a new service, get a pipeline, a repo, and baseline dashboards in minutes — not by filing a ticket to the platform team.
- **Shared vs. dedicated runner pools**: most teams use shared, autoscaled runner capacity; a few high-volume or specialized teams (e.g., needing GPU runners, or with strict isolation requirements) get dedicated pools. Cost-attribute usage per team/pipeline so the platform team can see which teams are driving spend.
- **Escape hatches with guardrails**: teams can customize their pipeline beyond the golden path, but customizations that skip mandatory security gates should be structurally impossible (the gates live in a shared, centrally-owned stage that all pipelines inherit, not something each team's pipeline file can opt out of).

This is the section most candidates skip, and it's usually where a strong answer is differentiated from an average one — an average answer stops at "here's a pipeline diagram."

## Step 4 — Security gates

Shift-left, but in a specific, ordered way (cheapest/fastest checks first so failures are caught before expensive steps run):

1. **Secret scanning** on every commit/push (cheapest, fastest, highest signal-to-noise — catch a leaked credential before it's even built).
2. **SAST** (static analysis) during build — catches code-level vulnerabilities (injection, unsafe deserialization) before packaging.
3. **SCA** (software composition analysis) against the dependency manifest — flags known-CVE dependencies before they're baked into an image.
4. **Container/image scanning** post-build — catches vulnerable base image layers, not just app dependencies.
5. **Image signing** (cosign/Sigstore) after a successful build+scan — signs only artifacts that passed every prior gate, and the CD operator refuses to deploy unsigned images.
6. **Policy as code** (OPA/Gatekeeper/Kyverno) at admission time in the cluster — a last-line enforcement layer independent of the pipeline (defense in depth: even if a pipeline is misconfigured, the cluster itself won't admit a policy-violating workload).

Call out explicitly: **new policies should ship in warn/audit mode first**, with a deprecation window before they start blocking — rolling out a new blocking gate to 500 engineers' pipelines overnight without warning is how a platform team becomes disliked and gets routed around.

## Step 5 — Multi-team scaling considerations

- **Ownership model**: a central platform team owns the golden-path templates, shared runners, and mandatory security gates; individual teams own their service-specific pipeline logic and deployment cadence. This is the standard "platform team enables, product teams own" split — state it explicitly, interviewers listen for it.
- **Queueing and autoscaling**: CI load is bursty (everyone pushes around the same working hours) — runners need to autoscale, and jobs need fair queueing so one team's huge test suite doesn't starve everyone else.
- **Build caching at scale**: shared, content-addressed caches (dependency caches, layer caches) keyed by lockfile/hash, not per-team silos — otherwise 500 engineers rebuild the same base layers redundantly, wasting compute and time.
- **Cost attribution**: tag pipeline runs and runner usage by team so cost isn't a black box — this becomes a real conversation at this scale and interviewers may probe for it.

## Step 6 — GitOps promotion flow

- Each environment (dev, staging, prod) has its desired state declared in a Git repo (or a clearly separated directory/branch per environment in a single config repo).
- Promotion between environments = a Git operation (merge/PR), **not a manual `kubectl apply` or a person SSHing anywhere**. This gives you a full audit trail: every change to what's running in prod is a reviewable, revertible Git commit.
- Lower environments (dev) can auto-promote on every successful build. Staging typically requires passing an automated test/soak gate. **Production promotion should require human approval** — this is both a safety gate and, in regulated contexts, a compliance requirement.
- The CD operator (Argo CD/Flux) continuously reconciles: if a target environment ever drifts from its declared Git state (someone manually edited a resource), the operator detects and can auto-correct or alert, depending on policy.
- Rollback = revert the Git commit, not "figure out the old config by hand" — this is the single biggest operational win GitOps gives you over ad hoc deployment scripts.

## Step 7 — Observability and rollback

- Deployment-specific metrics (error rate, latency, saturation) should be watched immediately post-deploy, scoped to the new version specifically (not just aggregate service health).
- Automated rollback on SLO breach for canary/progressive deployments removes the "someone has to notice and act fast at 2am" failure mode.
- Every deployment event should be correlated with metrics/logs/traces so "did this deploy cause the regression" is answerable in seconds, not by manually cross-referencing timestamps.

## Step 8 — Tradeoffs to state explicitly

A strong answer names tradeoffs unprompted rather than waiting to be asked:

- **Buy vs. build**: for a company this size, buying a managed CI (GitHub Actions/CircleCI) and CD/GitOps tooling (Argo CD is open-source but self-hosted; there are managed variants) is usually smarter than building custom orchestration — the differentiated value is in the golden paths and policies layered on top, not in reinventing a build scheduler.
- **Centralized platform team vs. federated ownership**: fully centralized becomes a bottleneck; fully federated (every team builds its own CI/CD) leads to inconsistency and duplicated security work. The golden-path model is a deliberate middle ground — name it as a tradeoff, not just a solution.
- **Security friction vs. velocity**: every gate added slows the pipeline and adds failure surface. Justify each gate's presence, and mention staged rollout (warn mode) as the mechanism for managing this tension.
- **Monorepo vs. polyrepo implications**: monorepo simplifies cross-service atomic changes and dependency consistency but complicates CI scoping (need path-based triggers so a change in one service doesn't rebuild everything); polyrepo isolates blast radius but multiplies the number of pipelines the platform team must template and maintain.

## Points to Remember

- Spend real time on requirements before architecture — this is the single most common thing candidates skip, and interviewers notice immediately when it's skipped.
- The org-scaling story (self-service, golden paths, ownership split) is what separates a platform design answer from a plain pipeline diagram — don't stop at the pipeline.
- Security gates should be ordered cheapest/fastest-first, and new gates should roll out in warn mode before they block anyone.
- Build once, promote the same artifact everywhere — say this explicitly, it signals real production experience.
- GitOps promotion = a Git operation with an audit trail and human approval before prod; rollback = revert a commit, not a manual scramble.
- Always close with tradeoffs stated unprompted (buy vs. build, centralized vs. federated, security vs. velocity) — this is what a senior-level answer sounds like.

## Common Mistakes

- Jumping straight to naming tools ("we'll use Jenkins and ArgoCD") before establishing requirements — sounds like memorized trivia, not judgment.
- Describing only a single team's pipeline and never addressing what changes at 500-engineer scale (self-service, ownership, queueing, cost attribution) — this is the actual point of the question.
- Forgetting security entirely until the interviewer prompts for it, instead of weaving gates into the architecture from the start.
- Presenting blue-green/canary/rolling as interchangeable without explaining what each optimizes for or costs — this reads as recall without understanding (see the CHEATSHEET for the comparison).
- Never stating a tradeoff unprompted — a design with zero acknowledged downsides reads as inexperienced, not confident.
- Treating GitOps promotion as "just another deploy step" instead of explaining *why* it exists (audit trail, drift correction, git-revert rollback).
