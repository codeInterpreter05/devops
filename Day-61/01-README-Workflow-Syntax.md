# Day 61 — GitHub Actions Fundamentals: Workflow Syntax

**Phase:** 2 – CI/CD & Security | **Week:** W10 | **Domain:** CI/CD | **Flag:** ⚡ Interview-critical

## Brief

GitHub Actions is the CI/CD engine baked directly into GitHub — no external system to wire up, no webhooks to babysit, just a `.github/workflows/*.yml` file and a `git push`. It's become the default CI choice for a huge share of open-source and increasingly enterprise projects, which means "read and write a GitHub Actions workflow" is now a baseline DevOps skill, not a specialty. Today is about the actual grammar of a workflow file: the `on`/`jobs`/`steps` skeleton everything else builds on.

This day is split into three files:

1. **This file** — the `on`/`jobs`/`steps` skeleton, triggers, runners, and how steps actually execute.
2. **[02-README-Contexts-And-Matrix.md](02-README-Contexts-And-Matrix.md)** — contexts (`github`, `env`, `secrets`, `runner`) and the matrix strategy for running one job across many configurations.
3. **[03-README-Reusable-Workflows-And-Caching.md](03-README-Reusable-Workflows-And-Caching.md)** — `workflow_call` reusable workflows, composite actions, and caching strategies.

## The anatomy of a workflow file

Every workflow lives at `.github/workflows/<name>.yml` and has three nested levels: **workflow → jobs → steps**.

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Node
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test
```

- **`on`** — the trigger(s). A workflow does nothing until an event fires: `push`, `pull_request`, `workflow_dispatch` (manual button), `schedule` (cron), `release`, `repository_dispatch` (external webhook trigger), or `workflow_call` (making this workflow callable from another one — covered in file 3).
- **`jobs`** — a workflow is one or more jobs. **By default, jobs run in parallel** on separate, fresh virtual machines (runners). If job B needs job A's output, you must say so explicitly with `needs: [job-a]` — there's no implicit ordering.
- **`runs-on`** — which runner image the job executes on: `ubuntu-latest`, `windows-latest`, `macos-latest`, or a self-hosted runner label (Day 62 covers self-hosted runners on EKS).
- **`steps`** — a job is a sequence of steps run **in order, on the same runner, sharing filesystem state** (unlike jobs, which start clean). A step is either:
  - `uses: <action>@<version>` — invoke a prebuilt, reusable action from the Marketplace or another repo.
  - `run: <shell command>` — execute a shell command directly (defaults to `bash` on Linux/macOS, `pwsh` on Windows).

## Why steps share state but jobs don't

Each **job** gets a brand-new VM (or container) — nothing from a previous job persists unless you explicitly pass it along via `needs` + `outputs`, or upload/download artifacts with `actions/upload-artifact` / `actions/download-artifact`. This is a deliberate isolation boundary: it lets GitHub run independent jobs on different runners (different OS, different hardware) and in parallel, but it means a common newcomer mistake is assuming a file built in one job is visible in the next — it isn't, unless you moved it.

**Steps**, by contrast, run sequentially on the *same* runner filesystem, so a file created by `run: npm run build` in one step is visible to the next step's `run: ls dist/` without any extra plumbing.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      artifact-name: ${{ steps.build.outputs.name }}
    steps:
      - id: build
        run: echo "name=app-$(date +%s)" >> "$GITHUB_OUTPUT"

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying ${{ needs.build.outputs.artifact-name }}"
```

Note the mechanism: a step writes `key=value` lines into the special file at `$GITHUB_OUTPUT` (an env var GitHub injects pointing at a temp file), and that becomes `steps.<id>.outputs.<key>` inside the same job, or `needs.<job>.outputs.<key>` in a downstream job that declares `needs: build` and the job re-exposes it through its own `outputs:` block. This two-hop mechanism (step output -> job output -> downstream job) trips people up constantly — a step output alone is **not** automatically visible outside its own job.

## Triggers in depth

```yaml
on:
  push:
    branches: [main, 'release/**']
    paths: ['src/**', '!src/**/*.md']   # ignore doc-only changes
  pull_request:
    types: [opened, synchronize, reopened]
  schedule:
    - cron: '0 2 * * *'      # 2 AM UTC daily — cron is always UTC
  workflow_dispatch:          # adds a "Run workflow" button in the UI
    inputs:
      environment:
        type: choice
        options: [staging, production]
        default: staging
```

- `paths`/`paths-ignore` filters prevent, e.g., a doc-only PR from triggering a full Docker build — a real cost and time saver on busy repos.
- `schedule` cron is evaluated in **UTC only** and is a "best-effort, may be delayed" trigger during high load — don't rely on it for time-critical jobs.
- `workflow_dispatch` inputs let you turn a workflow into a self-service tool (e.g., "deploy to prod" button) gated by GitHub's permission model, rather than a script someone runs locally with their own credentials.

## `runs-on` and job containers

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    container: node:20-alpine   # optional: run the whole job inside this container
    services:                    # sidecar containers, reachable by service name
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
        ports: ['5432:5432']
```

Using `container:` pins the exact OS/toolchain version the job runs in (independent of whatever GitHub bakes into `ubuntu-latest` that month — `ubuntu-latest` silently gets new tool versions periodically, which has broken pipelines before). `services:` spins up sidecar containers on the same Docker network as the job — the classic pattern for spinning up a throwaway Postgres/Redis for integration tests without touching a shared environment.

## Points to Remember

- Workflow structure is strictly `on` (trigger) → `jobs` (parallel by default, ordered only via `needs`) → `steps` (sequential, same runner, shared filesystem).
- Job-to-job data requires an explicit relay: step writes to `$GITHUB_OUTPUT` → job declares it under `outputs:` → downstream job reads via `needs.<job>.outputs.<key>`. Nothing crosses job boundaries implicitly except artifacts you explicitly upload/download.
- `paths`/`paths-ignore` filters save real CI minutes by skipping irrelevant triggers (e.g., docs-only changes).
- `schedule` cron is UTC-only and best-effort — GitHub explicitly disclaims exact-time guarantees under load.
- Pinning `container:` on a job gives you a stable, reproducible toolchain instead of relying on whatever `ubuntu-latest` happens to have installed today.

## Common Mistakes

- Assuming jobs run in the order they're written in the YAML — they run in **parallel** unless `needs` is specified, and forgetting this causes race conditions (e.g., a `deploy` job starting before `test` finishes).
- Expecting a file written in one job to exist in a later job without using artifacts — each job is a fresh VM with no shared filesystem.
- Writing to `echo "::set-output name=x::y"` — this old syntax is **deprecated and disabled** on current runners; the correct mechanism is appending to `$GITHUB_OUTPUT` (`echo "x=y" >> "$GITHUB_OUTPUT"`).
- Relying on `ubuntu-latest` for a toolchain-sensitive build (specific compiler/language version) and having it break months later when GitHub upgrades the underlying image — pin a `container:` or explicit setup-action version instead.
- Forgetting `paths`/`branches` filters, so every commit (including README typo fixes) triggers a full expensive pipeline.
