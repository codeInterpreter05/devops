# Day 78 — Quiz: Developer Experience in CI/CD

Try to answer without looking at your notes. Answers are at the bottom.

1. Why does a CI pipeline taking 25 minutes cost more than 25 minutes of a developer's actual time?
2. What's the difference between "inner loop" and "outer loop" checks, and which one should gate every single PR push?
3. Name three concrete techniques for shrinking CI runtime, and briefly explain the mechanism behind each.
4. What does `act` actually do under the hood, and name two things it cannot fully replicate compared to a real GitHub-hosted runner.
5. What are the two GitHub `pull_request` event actions you must both handle to implement a correct preview-environment lifecycle, and what happens if you only handle one?
6. Why is "we run CI on every push" not the same claim as "we do continuous integration," in the original/strict sense of the term?
7. What problem do feature flags solve that makes trunk-based development safe, and what is the "kill switch" pattern?
8. Name the four core DORA metrics and give the "elite performer" benchmark for each.
9. What is the counterintuitive core finding of the DORA research program regarding the relationship between deploy frequency and change failure rate?
10. Why is "PRs merged per day" a poor proxy for Deployment Frequency?
11. When does the MTTR clock start — and why does that specific choice matter?
12. **Interview question:** What are DORA metrics? What's a good benchmark for lead time for changes?

---

## Answers

1. Beyond the raw wait, a slow pipeline causes a **context switch** — the developer moves on to something else while waiting, and when the result finally comes back (often a failure), they have to mentally reload what they were doing before they can act on it. The real cost is the wait *plus* the resume-cold penalty, often paid multiple times across a day of iteration.
2. Inner loop = fast checks (lint, unit tests, type-check, quick build) that should run on every push, in a couple of minutes. Outer loop = slow, expensive checks (full e2e suites, load tests, cross-browser matrices) that should run on merge to main, on a schedule, or nightly — not block every PR. Only the inner loop should gate every push.
3. Any three of: caching dependencies keyed on a lockfile hash (skips reinstall on a cache hit); job matrix/test sharding (splits a long serial suite across parallel runners, trading runner-cost for wall-clock time); path filters (skip jobs entirely for irrelevant file changes, e.g., docs-only PRs); `concurrency` with `cancel-in-progress` (cancels a superseded run when a new push arrives, avoiding redundant queued runs); ordering cheap checks (lint/type-check) before expensive ones so trivial errors fail in seconds.
4. `act` parses your actual `.github/workflows/*.yml` and executes each job's steps inside Docker containers that approximate GitHub's hosted runner image, giving near-instant local feedback. It cannot fully replicate: GitHub context/secrets (must be passed explicitly via `-s`/`--secret-file`), and the exact hosted-runner image/toolset (the default `act` image is a slimmed-down approximation unless you opt into the much larger "full" image) — also, jobs depending on GitHub's real OIDC/`GITHUB_TOKEN` permission model or `services:` containers can behave differently.
5. `opened`/`synchronize`/`reopened` (to create/update the preview) and `closed` (to tear it down — which fires whether the PR was merged or simply closed). If you only handle the create side, closed PRs leave zombie namespaces/environments running indefinitely, quietly accumulating cost.
6. Continuous Integration, in its original sense, means frequently *integrating* changes into a shared trunk — daily or more often — so integration conflicts surface small and early. Running CI checks on every push doesn't help if branches themselves live for weeks before merging; you can have excellent per-push CI and still not be doing CI in the sense that matters, because the branching model (not the pipeline) determines integration frequency.
7. Feature flags decouple **deploy** (code reaches production, but inert/hidden) from **release** (a user actually experiences it) — this lets developers merge and deploy incomplete work into trunk safely, which is the prerequisite trunk-based development needs to avoid long-lived branches. The "kill switch" pattern wraps a risky new code path in a flag from day one so an incident responder can instantly disable it by flipping the flag, rather than needing a rollback deploy under time pressure.
8. Deployment Frequency (Elite: on-demand, multiple deploys/day); Lead Time for Changes (Elite: under 1 hour); Change Failure Rate (Elite: 0–15%); Time to Restore Service / MTTR (Elite: under 1 hour). (A fifth metric, Reliability, was added more recently and is SLO/org-defined.)
9. Speed and stability are not a tradeoff — elite performers deploy far more frequently *and* have a lower change failure rate than low performers, contradicting the older intuition that deploying less often and more cautiously is inherently safer.
10. It measures activity (how often code is merged), not outcome (how often code actually reaches production users) — it's easily inflated by many small, low-risk merges without reflecting any real improvement in delivery speed, and it says nothing about whether those merges were ever actually deployed.
11. It starts at **detection**, not at when the underlying defect was introduced or started silently affecting users. This matters because MTTR is meant to measure recovery speed specifically — conflating it with "time the bug existed undetected" would actually be measuring detection capability (a related but distinct metric, Mean Time to Detect), not recovery capability.
12. DORA metrics are four (now five) research-validated measures of software delivery performance from Google's DevOps Research and Assessment program: Deployment Frequency, Lead Time for Changes, Change Failure Rate, and Time to Restore Service (plus Reliability). A good benchmark for Lead Time for Changes at the elite performer tier is **under one hour** from commit to running successfully in production — worth also noting the counterintuitive finding that elite performers achieve this *and* a lower change failure rate simultaneously, since interviewers often follow up asking whether faster delivery trades off against stability.
