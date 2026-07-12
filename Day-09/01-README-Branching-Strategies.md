# Day 9 — Git Deep Dive II — Workflows: Branching Strategies

**Phase:** 0 – Foundation | **Week:** W2 | **Domain:** Git | **Flag:** —

## Brief

Two engineers can use identical Git commands and still ship at wildly different speeds, because *branching strategy* — not command knowledge — is what determines how fast code moves from a laptop to production. Today is about the two dominant models (GitFlow and trunk-based development), why the industry's center of gravity has shifted from the former to the latter, and the branch protection rules that let a team relax "just don't force-push to main" into something enforced by the platform instead of memory.

This day is split into three focused files:

1. **This file** — GitFlow vs. trunk-based development, and branch protection rules.
2. **[02-README-Git-Hooks-And-Conventional-Commits.md](02-README-Git-Hooks-And-Conventional-Commits.md)** — the hooks lifecycle, `pre-commit`, `husky`, Conventional Commits, and `commitlint`.
3. **[03-README-Signed-Commits-GPG.md](03-README-Signed-Commits-GPG.md)** — GPG-signed commits, verification, and SSH-based signing.

## GitFlow

GitFlow (Vincent Driessen, 2010) is a branching model built around **parallel long-lived branches**, each with a defined role:

- **`main`** (historically `master`) — always reflects what's in production. Every commit on it is a release.
- **`develop`** — the integration branch. Finished features land here first, ahead of a release.
- **`feature/*`** — branched off `develop`, merged back into `develop` when done. Can live for days or weeks.
- **`release/*`** — cut from `develop` when it's feature-complete for a release; only bug fixes and release-prep changes (version bumps, changelog) happen here. Merged into both `main` *and* `develop` when done.
- **`hotfix/*`** — branched directly from `main` to patch production urgently, then merged into both `main` and `develop`.

```
main:     ---o-----------------o---------------o------->
                \               \               /
release:         \        o------o-------------
                   \      /
develop:  ----o-----o----o----o-------o---------o------->
               \         /    \       /
feature/x:      o---o---o      o--o--o
```

**Why it emerged:** GitFlow was designed for software shipped in discrete, versioned releases — desktop apps, mobile apps, on-prem installers — where you genuinely need to freeze a release branch, stabilize it while `develop` keeps moving, and support multiple production versions (`v2.3.1` hotfix while `v2.4` is mid-development) at once.

**Why many teams now consider it heavyweight:**
- It assumes releases are infrequent and discrete. Teams doing continuous deployment (multiple production deploys per day) don't have a "release" as a distinct event — every merge to `main` *is* a release, so the `release/*` branch and the `develop`/`main` split add ceremony without adding safety.
- Long-lived `feature/*` and `develop` branches accumulate drift from `main`, so merges get larger and riskier the longer a branch lives — the opposite of what you want for fast, low-risk integration.
- Two integration points (`develop` and `main`) instead of one means two places for merge conflicts and two places state can silently diverge (a fix merged to `main` via hotfix but forgotten on `develop`, for example).
- It maps naturally onto teams that batch work into scheduled releases (versioned libraries, mobile app store submissions with review lag) — if that's not your delivery model, GitFlow's structure is solving a problem you don't have.

GitFlow is not "wrong" — it is still the right fit for shipping versioned artifacts with long-lived support windows (libraries, firmware, mobile). The mistake is defaulting to it for a service that deploys continuously.

## Trunk-based development

Trunk-based development (TBD) inverts the model: there is effectively **one long-lived branch** (`main`/`trunk`), and everyone integrates into it constantly.

- Branches, if used at all, are **short-lived** — hours, at most a day or two — created for a single small unit of work and merged back immediately.
- Direct small commits to `main` (with review via pull request, still typically required) are the norm rather than the exception.
- Because branches don't live long, merge conflicts stay small and are resolved close to when they're introduced, not weeks later against a codebase that's moved on.
- CI runs on every push to `main`, and `main` is expected to always be in a deployable state.

The mechanism that makes this safe for *continuous* deployment is **feature flags** (a.k.a. feature toggles): you merge incomplete or risky code into `main` behind a runtime flag that's off by default, so the code ships to production but isn't *active* for users. This decouples **merge** from **release** — you can integrate a half-finished feature into trunk today without exposing it, then flip the flag on later (for a percentage of users, an internal cohort, or everyone) without another deploy. Without feature flags, trunk-based development just means "merge broken code to main," which is why the two ideas are inseparable in practice — TBD without flags is reckless, GitFlow's isolation was partly compensating for not having them.

Feature flags also enable **trunk-based release patterns** that GitFlow can't do cleanly: canary releases, percentage rollouts, and instant kill-switches (flip the flag off instead of rolling back a deploy) — all without branching.

## GitFlow vs. trunk-based — the actual decision

| | GitFlow | Trunk-based |
|---|---|---|
| Branch lifetime | Days to weeks | Hours |
| Integration frequency | Batched, at feature/release completion | Continuous, multiple times a day |
| Release cadence it fits | Scheduled, versioned releases | Continuous deployment |
| Risk mitigation mechanism | Branch isolation (`develop` vs `main` vs `release`) | Feature flags + small, frequent merges + strong CI |
| Merge conflict size | Large (long divergence) | Small (short divergence) |
| Needs | Discipline around branch naming/merge order | Discipline around flag hygiene + CI speed/reliability |

The interview-safe framing: GitFlow trades integration speed for release isolation; trunk-based trades release isolation for integration speed, recovering safety through feature flags and fast CI instead of through branch structure. Most SaaS/cloud-native teams doing continuous deployment have moved to trunk-based for this reason — it's the default recommendation from the Trunk-Based Development site itself and from DORA's State of DevOps research, which correlates short-lived branches with higher deployment frequency and lower change failure rate.

## Branch protection rules

Branch protection is how you turn "please don't force-push to `main`" from a social norm into something the platform enforces. Both GitHub and GitLab expose roughly the same set of controls, with different names.

**Core protections (GitHub: Settings → Branches → Branch protection rules; GitLab: Settings → Repository → Protected branches / Merge request approvals):**

- **Require pull request before merging** — no direct pushes to the protected branch; all changes go through a PR/MR.
- **Require approvals** (GitHub: "Require a pull request before merging" → number of approvals; GitLab: "Approval rules") — at least N reviewers must approve. Often paired with **"Dismiss stale approvals on new commits"** so an approval doesn't silently cover code that changed after the review.
- **Require status checks to pass before merging** — CI (tests, linting, `shellcheck`, `commitlint`, build) must report green. GitHub lets you mark specific checks as **required**; a PR is blocked from merging until all of them succeed, and GitHub also offers **"Require branches to be up to date before merging"** so a check that passed against a stale base doesn't count.
- **Require signed commits** — rejects pushes containing unsigned commits (ties directly into GPG/SSH signing, covered in file 3).
- **Restrict who can push** — even with PRs required, you can further restrict merge rights to specific teams/roles.
- **Prevent force-push** (GitHub: "Do not allow force pushes"; GitLab: enabled by default on protected branches, can allow specific roles) — blocks `git push --force` from rewriting history on the protected branch, which would otherwise let anyone silently discard commits others have based work on.
- **Prevent deletion** — stops the protected branch itself from being deleted.
- **Require linear history** — disallows merge commits, forcing rebase-and-merge or squash-merge, so `main`'s history stays a straight line (easier `git bisect`, cleaner `git log`).
- **Require conversation resolution before merging** — blocks merging until every open PR review comment thread is marked resolved.

**Why these map to real incidents, not bureaucracy:**
- No-force-push + no-direct-push together are what actually make "PR review happened" a guarantee rather than a convention — without them, a required review can be bypassed by simply pushing straight to `main`.
- Required status checks are what make CI *matter*. CI that runs but doesn't gate the merge button is advisory only; a required check is enforcement.
- "Require branches up to date" closes a real gap: two PRs can each pass CI individually against an older `main`, but together introduce a conflict or breakage only visible once both land — requiring the branch to be current before merge (or using a merge queue) catches this.

## Points to Remember

- GitFlow: `main` + `develop` + `feature/*` + `release/*` + `hotfix/*`, built for scheduled/versioned releases with long support windows.
- Trunk-based: one main branch, short-lived branches (hours), continuous integration; feature flags decouple *merging* code from *releasing* it to users, which is what makes it safe for continuous deployment.
- The industry trend (SaaS, cloud-native, DORA research) favors trunk-based for services that deploy continuously; GitFlow still fits versioned artifacts like libraries and mobile apps.
- Branch protection turns social conventions ("don't force-push main") into platform-enforced rules: required PRs, required approvals, required status checks, no force-push, no deletion, optionally required signed commits and linear history.
- A required status check only has teeth if "require branches to be up to date" (or a merge queue) is also on — otherwise two individually-green PRs can still break `main` together.

## Common Mistakes

- Adopting GitFlow by default for a service that deploys many times a day — the `develop`/`release` split adds merge overhead and a second integration branch to keep in sync, solving a release-cadence problem the team doesn't have.
- Doing "trunk-based development" as just "we don't use branches" without feature flags — that's merging unfinished/risky code straight to production-tracking `main`, which is the exact failure mode branch isolation was protecting against.
- Enabling "require status checks" without also requiring the branch be up to date (or using a merge queue), letting two mutually-incompatible PRs both merge cleanly in sequence.
- Turning on branch protection but leaving an admin/owner bypass enabled ("Include administrators" unchecked on GitHub) — protection rules that exempt the people most likely to fat-finger a force-push provide a false sense of safety.
- Confusing "require pull request reviews" with "require status checks" — a human approval says nothing about whether the code compiles, passes lint, or passes tests; you need both gates, not one standing in for the other.
