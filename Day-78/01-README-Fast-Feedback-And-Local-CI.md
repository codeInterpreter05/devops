# Day 78 — Developer Experience in CI/CD: Fast Feedback Loops & Local CI

**Phase:** 2 – CI/CD & Security | **Week:** W13 | **Domain:** CI/CD | **Flag:** —

## Brief

A pipeline that's *correct* but slow gets routed around — developers push straight to a branch that skips checks, disable flaky tests instead of fixing them, or batch five days of work into one PR because "running CI takes forever anyway." Developer Experience (DX) in CI/CD is the discipline of treating pipeline speed and local reproducibility as first-class product requirements, not an afterthought bolted onto a "working" pipeline. This matters for interviews because platform/DevOps engineers are increasingly evaluated on organizational leverage — "did you make 200 engineers 10% faster" is a bigger story than "I wrote a working YAML file" — and fast feedback loops are the highest-leverage, most measurable way to do that.

This day is split into three files:

1. **This file** — why feedback speed matters and how to get it locally with `act`, before you ever push.
2. **[02-README-Preview-Environments-Trunk-Based-Dev.md](02-README-Preview-Environments-Trunk-Based-Dev.md)** — PR preview environments, trunk-based development, and feature-flag-driven CI.
3. **[03-README-DORA-Metrics.md](03-README-DORA-Metrics.md)** — measuring whether any of this actually worked, using DORA metrics.

## Why "under 5 minutes" is the number that matters

There's a well-known result (formalized loosely by Google's own internal build-time research, and echoed in *Accelerate*): once feedback on a change takes longer than a few minutes, developers stop waiting synchronously and **context-switch** to something else. The cost isn't the wait itself — it's the cost of resuming: reloading mental state about what you were doing, discovering the failure 20 minutes later after you've moved on, and now debugging cold instead of hot. A CI run that takes 25 minutes doesn't cost you 25 minutes; it costs you the wait *plus* the context-switch tax on both ends, often several times over across a day of iteration.

The concrete techniques that get you under 5 minutes:

- **Split "inner loop" from "outer loop."** Inner loop = the fast checks that run on every push (lint, unit tests, type-check, a quick build) — these should be seconds to a couple of minutes. Outer loop = slow, expensive checks (full end-to-end suites, load tests, cross-browser matrices) — run these on a schedule, on merge to main, or nightly, not blocking every single PR push.
- **Cache aggressively and correctly.** Dependency installs (`node_modules`, pip wheels, Maven `.m2`, Go module cache) rarely change between commits — cache them keyed on the lockfile hash so a cache hit skips the install entirely:

  ```yaml
  - uses: actions/cache@v4
    with:
      path: ~/.npm
      key: npm-${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}
      restore-keys: |
        npm-${{ runner.os }}-
  ```

  The `restore-keys` fallback matters: if there's no exact match (lockfile changed), you still get a *close* cache to build incrementally from instead of a fully cold run.

- **Parallelize with a job matrix and test sharding**, rather than one long serial job:

  ```yaml
  jobs:
    test:
      strategy:
        matrix:
          shard: [1, 2, 3, 4]
      steps:
        - run: npm test -- --shard=${{ matrix.shard }}/4
  ```

  Four shards running concurrently on four runners turn a 20-minute test suite into a ~5-minute one — you're trading runner-minutes (cost) for wall-clock time (developer feedback speed), which is almost always the right trade for the inner loop.

- **Skip irrelevant jobs with path filters.** A docs-only PR shouldn't trigger a full backend test matrix:

  ```yaml
  on:
    pull_request:
      paths:
        - 'src/**'
        - 'package-lock.json'
  ```

- **Cancel superseded runs** so a rapid string of pushes to the same PR doesn't queue up five redundant, resource-competing runs:

  ```yaml
  concurrency:
    group: ci-${{ github.ref }}
    cancel-in-progress: true
  ```

- **Fail fast on the cheapest checks first.** Order jobs/steps so lint and type-check (seconds) run before the expensive test suite (minutes) — a syntax typo shouldn't cost you a 10-minute wait to discover.

## Local CI with `act` — feedback before you even push

[`act`](https://github.com/nektos/act) runs your GitHub Actions workflows **locally**, inside Docker containers, by parsing `.github/workflows/*.yml` directly. Mechanically: it maps each `runs-on: ubuntu-latest` to a Docker image (by default a mid-sized image tagged `catthehacker/ubuntu:act-latest` that approximates GitHub's hosted runner) and executes your job's steps as if it were the real runner — including `uses:` actions, which it pulls and runs the same way GitHub does.

```bash
# Install (macOS)
brew install act

# Run every workflow triggered by `push`
act push

# Run a specific job
act -j build

# Simulate a pull_request event
act pull_request

# See what would run without executing it
act -n
```

Why this matters for DX: instead of "push → wait in the Actions queue → wait for the runner to boot → wait for the job → read the log → fix a YAML typo → repeat," you get a local iteration loop measured in seconds, entirely off GitHub's infrastructure and queue.

**Where `act` genuinely helps:**
- Catching YAML syntax errors and step-ordering mistakes before they cost you a real CI slot.
- Iterating on a new workflow or a tricky shell step without spamming your commit history with "fix ci" commits.
- Verifying a Dockerfile-based build step actually builds, fast, with local caching.

**Where `act` cannot fully replace the real runner** (know this cold — it's the practical limitation interviewers probe for):
- **GitHub context and secrets** aren't automatically available — you pass them explicitly via `--secret-file .secrets` or `-s KEY=value`, and anything that depends on GitHub's actual OIDC token, `GITHUB_TOKEN` permissions model, or repo-specific API behavior won't behave identically.
- **Runner image differences.** The default `act` image is a slimmed-down approximation of GitHub's hosted runner, not a byte-for-byte copy — a step that depends on a tool preinstalled on the real runner may fail locally unless you use the larger, closer image (`-P ubuntu-latest=catthehacker/ubuntu:full-latest`, multi-GB download) or install the tool as an explicit step.
- **Services and Docker-in-Docker.** Jobs using `services:` (e.g., a Postgres container for integration tests) or actions that themselves need to talk to Docker require `act` to mount the host Docker socket, which has its own quirks and isn't identical to the hosted runner's environment.
- **Self-hosted-runner-specific or GitHub-hosted-runner-specific steps** (e.g., anything reading runner-specific pre-installed tool versions) can silently diverge.

A `.actrc` file in the repo root lets you pin sane defaults so every teammate gets the same local behavior without memorizing flags:

```
-P ubuntu-latest=catthehacker/ubuntu:act-latest
--container-architecture linux/amd64
```

The practical rule: **use `act` to catch structural and logic errors fast, but still trust the real CI run as the final source of truth** — especially for anything security-, secrets-, or permissions-related.

## Points to Remember

- Feedback under ~5 minutes keeps developers in a synchronous wait state; beyond that, they context-switch, and failures get discovered — and debugged — cold, much later.
- Split inner loop (fast: lint, unit test, type-check, on every push) from outer loop (slow: e2e, load tests, on merge/schedule) — don't force every PR to pay the outer loop's cost.
- Cache keyed on the lockfile hash, parallelize with a job matrix/test sharding, filter irrelevant paths, and cancel superseded runs with `concurrency` — these four techniques alone usually get a slow pipeline back under budget.
- `act` gives near-instant local feedback on workflow structure and step logic by running your actual `.github/workflows/*.yml` in Docker, but it approximates the runner environment — it does not fully replicate GitHub context, secrets, or the exact hosted-runner image.
- Order cheap checks (lint, type-check) before expensive ones (full test suite) so trivial mistakes fail in seconds, not minutes.

## Common Mistakes

- Treating "add more parallelism" as a free lunch without noticing it multiplies runner cost — sharding a suite across 10 runners for a 30-second win is often not worth it; measure the actual time saved against the actual dollar cost.
- Caching dependencies keyed only on `runner.os` without including the lockfile hash — this serves a **stale** cache after a dependency bump, causing confusing "works locally, broken in CI" bugs that are actually a cache problem, not a code problem.
- Assuming a workflow that passes under `act` will definitely pass on GitHub's real runners — skipping the final real-CI run for anything touching secrets, OIDC-based cloud auth, or GitHub-API-specific behavior.
- Not adding `concurrency: cancel-in-progress`, so a developer pushing three quick fixups in a row queues three full redundant runs, burning both runner-minutes and making the Actions UI confusing to read.
- Putting the expensive end-to-end suite in the same required-status-check gate as fast unit tests, so every PR pays outer-loop cost even for a one-line docs fix that a path filter should have skipped entirely.
