# Day 62 — Advanced GitHub Actions: Concurrency Control & Interactive Debugging

**Phase:** 2 – CI/CD & Security | **Week:** W10 | **Domain:** CI/CD | **Flag:** None

## Brief

Two practical skills round out "advanced" GitHub Actions usage: preventing pipelines from stepping on each other when multiple runs overlap (concurrency control), and actually getting inside a failing runner to poke around instead of guessing from logs (`tmate` debugging). Both come up constantly once a pipeline is real — a busy repo with many rapid pushes and deploy jobs racing each other is a "when," not "if," problem.

## Concurrency control — preventing overlapping runs from colliding

By default, every trigger event spawns a brand-new, independent workflow run — pushing three commits to the same PR in quick succession queues (or runs in parallel) three separate CI runs. For most CI (test/lint) this is wasteful but harmless. For **deploys**, it's dangerous: two deploy jobs racing against the same environment can interleave in ways that corrupt state (e.g., two `terraform apply` runs against the same state file, or two `kubectl apply` runs targeting the same resources with different versions).

```yaml
concurrency:
  group: deploy-production
  cancel-in-progress: false
```

- **`group`** — workflow runs sharing the same group value are serialized: only one runs at a time; others queue.
- **`cancel-in-progress`** — if `true`, a new run in the same group **cancels** whatever's currently running instead of waiting for it. Correct for CI (a new push supersedes an old, now-stale test run); dangerous for deploys (cancelling a deploy mid-flight can leave infrastructure in a half-applied state).

```yaml
# Typical CI pattern: cancel stale runs on new pushes to the same PR
concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

# Typical deploy pattern: queue, never cancel mid-deploy
concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: false
```

The `group` expression is the whole design decision. `${{ github.workflow }}-${{ github.ref }}` scopes concurrency **per branch/PR** — pushes to different PRs don't block each other, only rapid pushes to the *same* PR do. A group of just `production-deploy` (no ref) scopes concurrency **globally across the entire environment**, which is what you want for a deploy target that can only sensibly do one thing at a time regardless of which branch triggered it.

Concurrency can be set at the workflow level (applies to the whole run) or at the job level (only that job is serialized, other jobs in the same workflow run freely) — job-level scoping is often the right call when only the deploy job needs serialization but test/lint should still run freely per-push.

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps: [...]   # no concurrency restriction — runs freely

  deploy:
    needs: test
    concurrency:
      group: deploy-production
      cancel-in-progress: false
    runs-on: ubuntu-latest
    steps: [...]
```

## Interactive workflow debugging with `tmate`

Sometimes a failure only reproduces in the actual GitHub runner environment — a subtle PATH difference, a missing system library, an env var that behaves differently than local. Re-adding `echo` statements and re-pushing is slow. `tmate` (via `mxschmitt/action-tmate`) opens an actual interactive SSH session into the live runner, mid-workflow, so you can poke around exactly as the failure is happening.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run build
        run: ./build.sh

      - name: Debug session on failure
        if: failure()
        uses: mxschmitt/action-tmate@v3
        timeout-minutes: 15
```

- `if: failure()` — only opens the debug session when a previous step actually failed, so it doesn't slow down/expose every normal run.
- The step's log output prints an SSH connection string (`ssh <random>@nyc1.tmate.io`) — you paste that into your terminal and land inside the exact failed runner's shell, with the failed job's filesystem state intact, able to run any command to investigate.
- **Always set a `timeout-minutes`** — without one, the job (and the runner minutes it consumes) can hang indefinitely waiting for someone to connect, burning CI budget and holding up any concurrency group it belongs to.
- You can also force a debug session unconditionally (drop the `if: failure()`) for a one-off exploratory run, or gate it behind a `workflow_dispatch` boolean input so it's opt-in rather than always active.

```yaml
on:
  workflow_dispatch:
    inputs:
      debug_enabled:
        type: boolean
        default: false

steps:
  - name: Debug session
    if: ${{ github.event.inputs.debug_enabled == 'true' }}
    uses: mxschmitt/action-tmate@v3
```

### Security consideration for `tmate`

A live interactive shell into a runner is, by definition, a powerful access primitive — anyone who can read the workflow log while the session is open can grab the connection string. Restrict `tmate` steps to trusted maintainers (never enable it unconditionally on workflows triggered by external/fork PRs), and never leave a debug session wired to run on every single CI execution in a shared or public repo.

GitHub's own **re-run with debug logging** (`ACTIONS_STEP_DEBUG` / `ACTIONS_RUNNER_DEBUG` secrets set to `true`, or the "Enable debug logging" checkbox on re-run) is a lighter-weight first step before reaching for a full interactive `tmate` session — it surfaces much more verbose log output without granting shell access.

## Points to Remember

- `concurrency.group` defines what "overlapping" means — include `github.ref` to scope per-branch/PR, or omit it to scope globally across an entire environment/target.
- `cancel-in-progress: true` is right for CI (stale runs are wasted work); `false` is right for deploys (cancelling mid-flight risks half-applied state).
- Concurrency can be scoped at workflow level or per-job — use per-job when only the deploy step needs serialization.
- `tmate` debug steps should be gated (`if: failure()` or a manual `workflow_dispatch` toggle) and always carry a `timeout-minutes` to avoid runaway sessions burning CI minutes.
- Debug logging (`ACTIONS_STEP_DEBUG`) is a cheaper, non-interactive first step before reaching for a full `tmate` shell.

## Common Mistakes

- Setting `cancel-in-progress: true` on a deploy workflow "to match the CI pattern," causing a newer push to cancel an in-progress deploy and leave infrastructure partially applied.
- Choosing a concurrency `group` that's too broad (e.g., just the workflow name, no `github.ref`), accidentally serializing unrelated PRs' CI runs against each other and creating a bottleneck.
- Leaving a `tmate` debug step active unconditionally on a workflow that fork PRs can trigger — handing shell access on your infrastructure to any external contributor whose PR fails a check.
- Forgetting `timeout-minutes` on a `tmate` step — an abandoned debug session can hold a runner (and a concurrency group, blocking all subsequent runs) for hours.
- Reaching for `tmate` before trying `ACTIONS_STEP_DEBUG`/verbose re-run logging, when the answer was visible in the extra log detail all along.
