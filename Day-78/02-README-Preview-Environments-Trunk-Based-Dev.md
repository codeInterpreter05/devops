# Day 78 — Developer Experience in CI/CD: Preview Environments, Trunk-Based Dev & Feature Flags

**Phase:** 2 – CI/CD & Security | **Week:** W13 | **Domain:** CI/CD | **Flag:** —

## Brief

These three practices — ephemeral per-PR preview environments, trunk-based development, and feature-flag-driven releases — solve the same underlying problem from three different angles: **how do you let many people ship continuously into one codebase without long-lived branches, shared-staging contention, or "big scary release" anxiety?** They show up together constantly because they're mutually reinforcing: trunk-based dev only works if you have a safe way to hide unfinished work (feature flags) and a safe way to validate a change before merging (preview environments). Understanding them as one connected system, not three isolated tools, is what separates "I've heard of feature flags" from actually being able to design a team's delivery workflow.

## PR preview environments — validate before you merge, not after

A preview environment is a **full, isolated, ephemeral deployment of your application for a single PR**, spun up automatically when the PR opens and torn down automatically when it closes. The point: instead of merging to a shared staging environment (where five people's half-finished changes collide and "who broke staging" becomes a recurring Slack thread), every PR gets its own throwaway environment that only reflects that PR's changes.

**Mechanically, on Kubernetes**, this usually means a namespace-per-PR pattern:

```yaml
# .github/workflows/preview.yml
name: PR Preview
on:
  pull_request:
    types: [opened, synchronize, reopened, closed]

jobs:
  deploy-preview:
    if: github.event.action != 'closed'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Deploy preview namespace
        run: |
          NS="pr-${{ github.event.pull_request.number }}"
          kubectl create namespace "$NS" --dry-run=client -o yaml | kubectl apply -f -
          helm upgrade --install "$NS" ./chart \
            --namespace "$NS" \
            --set image.tag=${{ github.sha }} \
            --set ingress.host="$NS.preview.example.com"
      - name: Comment preview URL on PR
        uses: marocchino/sticky-pull-request-comment@v2
        with:
          message: "🚀 Preview deployed: https://pr-${{ github.event.pull_request.number }}.preview.example.com"

  teardown-preview:
    if: github.event.action == 'closed'
    runs-on: ubuntu-latest
    steps:
      - name: Delete preview namespace
        run: kubectl delete namespace "pr-${{ github.event.pull_request.number }}" --ignore-not-found
```

The key trigger detail: `pull_request` fires on `opened`/`synchronize` (new commits pushed) for the deploy job, and separately on `closed` (merged **or** just closed without merging) for the teardown job — miss the teardown trigger and you accumulate zombie namespaces that quietly burn cluster resources and cost.

Tools that do this pattern for you rather than hand-rolling it: **Argo CD's PR generator** (part of ApplicationSets — watches open PRs on a repo and materializes/destroys an Argo CD Application per PR automatically), or platform-as-a-service equivalents like Vercel/Netlify/Render for frontend-heavy apps, which give you this for free with zero YAML.

**Real risks to design around, not just accept:**
- **Cost creep** — dozens of forgotten preview environments running indefinitely if the teardown hook ever fails silently. Add a scheduled sweep job (`kubectl get ns -l type=preview` older than N days → delete) as a safety net.
- **Secrets exposure** — a preview environment is often reachable at a guessable/public URL; never point it at production data or real secrets. Use synthetic/seeded data and scoped-down credentials.
- **Drift from production topology** — a preview environment that's a single-pod simplification of a multi-service production system will validate the wrong things; keep it as close to real topology as cost allows for anything you actually rely on the preview to catch.

## Trunk-based development — the actual prerequisite for continuous integration

Trunk-based development (TBD) means: every developer works off very short-lived branches (ideally under a day, rarely more than two) and merges directly into `main` (the "trunk") frequently — as opposed to GitFlow-style long-lived `feature/*` or `release/*` branches that diverge from `main` for days or weeks.

This is worth stating plainly because it's a common interview trap: **"we run CI on every push" does not mean you're doing continuous integration if branches live for two weeks before merging.** Continuous Integration, the actual practice the term originally described, means frequently *integrating* your changes into the shared trunk — every day, ideally many times a day — so integration conflicts surface small and early instead of as one enormous, dreaded merge at the end of a long-lived branch. A pipeline can be technically excellent and you can still not be "doing CI" in the sense that matters, if your branching model defeats the entire point.

**What makes TBD viable in practice:**
- A **fast, trustworthy CI pipeline** gating every merge to `main` (this is why Day 78's first file matters — nobody merges frequently into a trunk that takes 25 minutes and is flaky to validate against).
- **Feature flags** to merge incomplete work safely (see below) — the alternative, an unmergeable half-finished branch living for weeks, is exactly the long-lived-branch problem TBD exists to avoid.
- **Branch protection rules** requiring the CI status checks to pass and (usually) at least one review before merge, so "frequent merges" doesn't mean "frequent breakage."
- Optionally, **short-lived release branches cut from trunk at deploy time** (not developed on) for teams needing a stabilization window — still fundamentally trunk-based, just with a thin cut-and-ship step layered on top.

## Feature flags + CI: decoupling "deploy" from "release"

A feature flag (feature toggle) is a runtime switch — typically backed by a service like **LaunchDarkly**, or a simpler homegrown config table — that decides whether a code path executes, independent of what's actually deployed. This is the mechanism that makes trunk-based development safe: you can merge and deploy incomplete or risky code to production *dark* (flag off, invisible to users), and separately decide, later and independently of any deploy, when to actually turn it on for some or all users.

**Integrating flags into the pipeline, concretely:**
- **Flags as code.** Define flags via the LaunchDarkly Terraform provider (or equivalent API) so flag creation/deletion is reviewed and versioned the same way infrastructure is, rather than living only as ad hoc clicks in a dashboard:

  ```hcl
  resource "launchdarkly_feature_flag" "new_checkout_flow" {
    project_key = "web-app"
    key         = "new-checkout-flow"
    name        = "New checkout flow"
    variation_type = "boolean"
    variations {
      value = "true"
    }
    variations {
      value = "false"
    }
  }
  ```

- **Progressive rollout tied to deploy confidence**, e.g., 5% of users → watch error rates and DORA-style change-failure signals → 25% → 100%, all without a second deploy — this is functionally a canary release implemented at the application layer instead of (or in addition to) the infrastructure layer (compare to a Kubernetes/Argo Rollouts traffic-split canary, which is the same idea enforced by the platform instead of app code).
- **Kill switch pattern** — wrap risky new code paths in a flag from day one specifically so an incident responder can flip it off instantly without a rollback deploy, which is dramatically faster than `git revert` → rebuild → redeploy under incident pressure.
- **Flag hygiene as a CI gate.** Stale flags are real technical debt — a flag left permanently at 100%/0% with no cleanup accumulates dead conditional branches nobody's confident to delete. Some teams add an automated check (a linter or a scheduled job querying the flag API) that flags-that-are-flags older than N days at 100% rollout and opens a ticket to remove the toggle and the dead code path.

## Points to Remember

- Preview environments validate a PR's actual deployed behavior *before* merge, in an isolated namespace/environment — the create/destroy lifecycle must be triggered symmetrically (`opened`/`synchronize` to deploy, `closed` to tear down) or you leak cost.
- Trunk-based development — short-lived branches, frequent merges to `main` — is the actual prerequisite for continuous integration in the original sense of the term; a fast CI pipeline alone doesn't make you "CI" if branches live for weeks.
- Feature flags decouple **deploy** (code reaches production, inert) from **release** (a user actually experiences the new behavior) — this is what makes merging incomplete work into trunk safe.
- Flags-as-code (e.g., via a Terraform provider) brings the same review/versioning discipline to flag changes that you'd expect for infrastructure changes.
- A flag's job isn't done at 100% rollout — cleanup (removing the flag and the dead old-code-path) is part of the definition of done, not an optional follow-up that never happens.

## Common Mistakes

- Forgetting the teardown half of a preview-environment workflow (only handling `opened`), leaving a growing pile of zombie namespaces/costs that nobody notices until a cloud bill spike.
- Calling a workflow "trunk-based" while actually running week-long feature branches with a squash-merge at the end — the branch *name* doesn't matter; the *lifetime* before integration is what determines whether you get TBD's actual benefit.
- Treating feature flags as free — leaving old flags in the codebase indefinitely creates a combinatorial mess of untested flag-state permutations and code that's effectively impossible to safely delete later.
- Pointing a preview environment at production data/secrets "just to test something real" — this turns a low-stakes, public-facing throwaway environment into a genuine data-exposure risk.
- Rolling out a feature flag to 100% and calling it done without ever removing the `if (flag)` branching and the old code path — six months later nobody remembers which branch is actually live, and the flag becomes an undocumented, silent single point of confusion during an incident.
