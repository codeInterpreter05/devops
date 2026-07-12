# Day 61 — GitHub Actions Fundamentals: Reusable Workflows, Composite Actions & Caching

**Phase:** 2 – CI/CD & Security | **Week:** W10 | **Domain:** CI/CD | **Flag:** ⚡ Interview-critical

## Brief

Once you own more than a couple of workflows, duplication becomes the enemy — the same "checkout, auth to AWS, build, push to ECR" block copy-pasted across ten repos means ten places to fix when the process changes. Reusable workflows and composite actions are GitHub Actions' two answers to "don't repeat yourself," and caching is the lever that turns a 20-minute pipeline into a 5-minute one — exactly the interview question attached to this day.

## Reusable workflows (`workflow_call`)

A reusable workflow is a normal workflow file that declares `workflow_call` as a trigger, making it callable **from another workflow** (not just from GitHub events directly).

```yaml
# .github/workflows/build-and-push.yml (the reusable workflow)
on:
  workflow_call:
    inputs:
      image-name:
        required: true
        type: string
      environment:
        required: false
        type: string
        default: staging
    secrets:
      registry-token:
        required: true
    outputs:
      image-digest:
        value: ${{ jobs.build.outputs.digest }}

jobs:
  build:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    outputs:
      digest: ${{ steps.push.outputs.digest }}
    steps:
      - uses: actions/checkout@v4
      - name: Build and push
        id: push
        run: |
          docker build -t "${{ inputs.image-name }}" .
          echo "digest=$(docker inspect --format='{{index .Id}}' "${{ inputs.image-name }}")" >> "$GITHUB_OUTPUT"
```

```yaml
# .github/workflows/ci.yml (the caller)
jobs:
  release:
    uses: ./.github/workflows/build-and-push.yml
    with:
      image-name: myapp:${{ github.sha }}
      environment: production
    secrets:
      registry-token: ${{ secrets.ECR_TOKEN }}
```

Key mechanics:
- The caller invokes it with `uses: ./path/to/workflow.yml` (same repo) or `uses: org/repo/.github/workflows/file.yml@ref` (cross-repo — pin to a tag/SHA in production, exactly like pinning an action).
- **Secrets do not automatically flow through** — you must explicitly pass each one under `secrets:`, unless you use the shorthand `secrets: inherit` to forward *all* the caller's secrets. `inherit` is convenient but expands the reusable workflow's blast radius — anyone who can call it gets access to everything the caller has, so it's a tradeoff between convenience and least-privilege.
- Reusable workflows can be nested up to 4 levels deep and can define their own `outputs` that bubble back to the caller — this is the standard pattern for a shared "build once, deploy everywhere" pipeline used across many microservice repos with one canonical definition.

### Reusable workflow vs. composite action — when to use which

| | Reusable workflow (`workflow_call`) | Composite action |
|---|---|---|
| Scope | Can contain multiple **jobs**, each with its own `runs-on` | Runs as a set of **steps inside a single existing job** |
| Can set its own runner? | Yes | No — inherits the calling job's runner |
| Can define secrets contract? | Yes, explicit `secrets:` block | No — reads whatever `env`/context the calling job already has |
| Best for | An entire pipeline stage (e.g., "the full test-build-scan-push flow") shared across repos | A reusable sequence of steps (e.g., "set up Node, restore cache, install deps") embedded inside a larger job |

## Composite actions

A composite action packages several `run`/`uses` steps into one callable unit that consumers reference as a single `uses:` step within their own job.

```yaml
# .github/actions/setup-node-app/action.yml
name: 'Setup Node App'
description: 'Checkout, setup Node, install deps with cache'
inputs:
  node-version:
    required: false
    default: '20'
runs:
  using: 'composite'
  steps:
    - uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        cache: 'npm'
    - run: npm ci
      shell: bash   # required on every 'run' step in a composite action
```

```yaml
# consumer workflow
steps:
  - uses: actions/checkout@v4
  - uses: ./.github/actions/setup-node-app
    with:
      node-version: '22'
```

Note the `shell: bash` requirement — unlike a normal workflow step (which infers a default shell), every `run` step inside a composite action must declare its shell explicitly, because composite actions are designed to be portable across different consuming contexts.

## Caching strategies — the actual 20-minutes-to-5-minutes lever

`actions/cache` (and the built-in `cache:` option on setup-* actions) avoids re-downloading/re-building the same dependencies on every run.

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'          # shorthand: caches based on package-lock.json hash automatically

# or the manual, general-purpose form:
- uses: actions/cache@v4
  with:
    path: |
      ~/.npm
      node_modules
    key: npm-${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      npm-${{ runner.os }}-
```

Mechanics that actually matter:
- **Cache key must include a hash of the lockfile** (`hashFiles('**/package-lock.json')`). If the lockfile changes, the key changes, so you get a cache miss and rebuild — correct, because dependencies actually changed. Using a static key (e.g., just `npm-cache`) means you'll keep restoring stale dependencies even after `package-lock.json` changes, silently masking dependency bumps.
- **`restore-keys` provides fallback partial matches** — if there's no exact key match, GitHub restores the most recent cache with a matching prefix, giving you a "close enough" cache (most deps still valid, a few new ones installed on top) instead of a full cold cache.
- Caches are scoped per-branch by default with cross-branch fallback to the base branch — a PR branch will fall back to `main`'s cache if it has no cache of its own yet, which is why first-run-on-a-new-branch is often still fast.
- GitHub's cache has a **10GB per-repo limit** with LRU eviction — bloated caches (whole `node_modules` plus build artifacts you don't need) push out other useful caches sooner.
- Docker layer caching is different from `actions/cache` — for Docker builds, use `docker/build-push-action` with `cache-from`/`cache-to: type=gha` to cache Docker layers specifically in GitHub's cache backend, which is what actually collapses image rebuild time.

```yaml
- uses: docker/build-push-action@v6
  with:
    context: .
    push: true
    tags: myregistry/app:${{ github.sha }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

### Concrete plan for cutting 20 minutes to 5

1. **Cache dependencies** (npm/pip/maven/Docker layers) keyed on the lockfile hash — usually the single biggest win.
2. **Parallelize independent jobs** (lint, unit test, build) instead of running them sequentially in one job.
3. **Use a matrix with sharding** to split a large test suite across N parallel runners.
4. **Skip unnecessary triggers** with `paths`/`paths-ignore` filters (file 1) so doc-only PRs don't run the full pipeline.
5. **Use `docker/build-push-action` with `cache-from/to: type=gha`** for layer caching instead of rebuilding every layer from scratch.
6. **Pin a `container:`/smaller base image** so setup steps (installing a toolchain) aren't repeated — or better, use a prebuilt image that already has the toolchain baked in.

## Points to Remember

- A reusable workflow can define multiple jobs and its own runner per job; a composite action is steps injected into the *caller's* existing job and runner.
- Secrets don't automatically propagate into a called reusable workflow — pass them explicitly, or use `secrets: inherit` and accept the wider blast radius that comes with it.
- Every `run:` step inside a composite action needs an explicit `shell:` — normal workflow steps don't need this because GitHub infers a default.
- Cache keys must incorporate a hash of the dependency manifest (lockfile) — a static key silently serves stale dependencies.
- `restore-keys` gives partial-match fallback; without it, any cache-key miss means a fully cold cache.
- Docker layer caching needs its own mechanism (`type=gha` in `docker/build-push-action`) — `actions/cache` alone doesn't optimize Docker builds well.

## Common Mistakes

- Using a static cache key (`key: npm-cache`) that never changes — dependencies silently go stale, or worse, the cache never actually reflects `package-lock.json` updates.
- Forgetting `shell: bash` on a `run:` step inside a composite action and getting a confusing "Required property is missing" error.
- Using `secrets: inherit` by default on every reusable workflow call "to save time," without realizing every caller now hands over its entire secret set — an unnecessary widening of blast radius if a workflow only needs one specific token.
- Not pinning cross-repo reusable workflow/action references to a tag or SHA (`uses: org/repo/.github/workflows/x.yml@main`) — a moving `@main` reference means the shared pipeline's behavior (and security posture) can change under you without any change on your end.
- Caching `node_modules` directly without also keying on OS (`runner.os`) when using a matrix across multiple operating systems — native binary modules built for Linux silently get restored (and fail) on a macOS runner.
- Treating caching as "free" and caching enormous build artifacts that rarely change together with fast-changing ones under one key, causing frequent invalidation of the whole blob instead of granular, independently-keyed caches.
