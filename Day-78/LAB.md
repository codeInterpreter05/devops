# Day 78 — Lab: Developer Experience in CI/CD

**Goal:** Speed up a real CI pipeline, stand up a throwaway PR preview environment, and — the assigned hands-on activity — actually measure your own pipeline's DORA metrics from real data instead of estimating them.

**Prerequisites:** A GitHub repo with at least one working GitHub Actions workflow (any language is fine — use a small sample app if you don't have one handy). Docker installed locally (required for `act`). Optional: a local Kubernetes cluster (`kind` or `minikube`) if you want to do the preview-environment lab against real Kubernetes rather than simulate it.

---

### Lab 1 — Install and run `act` locally

1. Install `act`:
   ```bash
   brew install act        # macOS
   # or: https://github.com/nektos/act#installation for Linux/Windows
   ```
2. In a repo with an existing `.github/workflows/*.yml`, run:
   ```bash
   act -n              # dry run — see what would execute
   act push            # actually run the push-triggered workflow(s) locally
   ```
3. Deliberately introduce a YAML mistake (bad indentation, a typo'd `uses:` version) and confirm `act` catches it in seconds, before you'd have pushed and waited on a real runner.
4. Create a `.actrc` in the repo root pinning a runner image and architecture so results are reproducible for anyone on the team:
   ```
   -P ubuntu-latest=catthehacker/ubuntu:act-latest
   --container-architecture linux/amd64
   ```

**Success criteria:** You can run any job from your workflow locally with `act -j <job-name>` and get a pass/fail result without touching GitHub's Actions queue.

---

### Lab 2 — Shrink your pipeline's runtime, and measure the before/after

1. Time your current pipeline's slowest job from trigger to completion (check the Actions UI's duration, or time it manually).
2. Add dependency caching keyed on your lockfile hash (adjust path/key for your stack):
   ```yaml
   - uses: actions/cache@v4
     with:
       path: ~/.npm
       key: npm-${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}
   ```
3. Add a `concurrency` block to cancel superseded runs:
   ```yaml
   concurrency:
     group: ci-${{ github.ref }}
     cancel-in-progress: true
   ```
4. If your test suite is large enough to matter, split it into a 2-way matrix shard and compare total wall-clock time against the single-job baseline.
5. Push twice in quick succession and confirm the first run gets cancelled automatically instead of both running to completion.

**Success criteria:** A documented before/after duration for your slowest job, plus a screenshot or log showing a superseded run being auto-cancelled.

---

### Lab 3 — Stand up (and tear down) a PR preview environment

1. If you have a `kind` cluster available, create a minimal deploy workflow that creates a namespace named after the PR number and applies a simple manifest (a single `Deployment` + `Service` is enough — this is about the lifecycle pattern, not the app):
   ```yaml
   on:
     pull_request:
       types: [opened, synchronize, reopened, closed]
   jobs:
     preview:
       if: github.event.action != 'closed'
       runs-on: ubuntu-latest
       steps:
         - run: kubectl create namespace pr-${{ github.event.number }} --dry-run=client -o yaml | kubectl apply -f -
     teardown:
       if: github.event.action == 'closed'
       runs-on: ubuntu-latest
       steps:
         - run: kubectl delete namespace pr-${{ github.event.number }} --ignore-not-found
   ```
2. Open a real PR against your test repo, confirm the namespace gets created (`kubectl get ns`), then close the PR and confirm it gets deleted.
3. If you don't have cluster access in your CI runner, simulate the same lifecycle logic locally with `act pull_request` and manually flip the event's `action` field in a test payload to verify both branches of your `if` conditions fire correctly.

**Success criteria:** You can point to a namespace (or equivalent isolated resource) that was created on PR open and confirm it was deleted on PR close — no leftover resources.

---

### Lab 4 — The core hands-on activity: measure your pipeline's DORA metrics

1. Pick a real repo with some deployment history (your own project repo, or this learning repo's commit history as a stand-in if nothing else is deployed).
2. **Deployment Frequency**: count production deployment events (or, as a proxy, merges to `main` if you have no real deploy events) over the last 30 days: `git log --since="30 days ago" --oneline main | wc -l`. State the frequency as "X per week."
3. **Lead Time for Changes**: pick 5 recent merged PRs. For each, compute the delta between the first commit's timestamp and the merge commit's timestamp:
   ```bash
   git log --format='%H %aI' <first-commit-sha> -1
   git log --format='%H %aI' <merge-commit-sha> -1
   ```
   Average the five deltas.
4. **Change Failure Rate**: review your last 20 deploys/merges to `main` — how many required a follow-up revert, hotfix, or rollback commit within the next 24 hours? `failed / total`.
5. **MTTR**: if you have any incident/bug-fix history, find a bug-fix PR, note when the issue was reported/detected versus when the fix was merged and deployed — compute the delta. If you have no real incident, simulate one: intentionally break something in a test branch, "detect" it, and time your own fix-to-deploy cycle.
6. Write up all four numbers in a short `DORA-METRICS.md` and classify your own pipeline as Elite/High/Medium/Low against the benchmarks from today's third README.

**Success criteria:** A written report with all four DORA metrics computed from real data (or clearly-labeled simulated data), plus a one-paragraph honest assessment of which performer tier your pipeline currently falls into and the single biggest lever to move it up a tier.

---

### Cleanup

```bash
kubectl delete namespace pr-<number> --ignore-not-found   # if any preview namespaces remain
rm -f .secrets                                             # if you created one for local act runs — never commit this file
```

### Stretch challenge

Wire up a minimal Grafana/Prometheus dashboard (or even a simple script writing to a CSV) that recomputes Deployment Frequency and Change Failure Rate automatically from the GitHub API on a schedule, instead of the one-time manual calculation from Lab 4 — this is the actual shape of what "measure DORA metrics" means in a real organization.
