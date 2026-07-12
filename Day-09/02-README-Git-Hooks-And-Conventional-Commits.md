# Day 9 — Git Deep Dive II — Workflows: Git Hooks and Conventional Commits

**Phase:** 0 – Foundation | **Week:** W2 | **Domain:** Git | **Flag:** —

## Brief

Code review catches bugs; it rarely catches "the shell script has an unquoted variable" or "this commit message can't be parsed by our release tooling" — those are exactly the class of mechanical, repetitive checks that should never depend on a human remembering to look. Git hooks are the mechanism for running checks automatically at specific points in the Git workflow, and `pre-commit` / `husky` are the tooling that makes hooks *shareable and versioned* instead of a personal `.git/hooks/pre-commit` script that only exists on one laptop. Conventional Commits gives those hooks something structured to enforce on the commit message itself, which is what unlocks automated changelogs and semantic versioning.

## The Git hooks lifecycle

Git hooks are scripts Git runs automatically at specific points in its own lifecycle. They live in `.git/hooks/` as plain executable scripts (any language — shell, Python, Node — as long as the file is executable and has the right name, no extension). Because `.git/` is never committed, **raw hooks are local-only by default** — this is the exact problem `pre-commit` and `husky` solve (see below).

**Client-side hooks** (run on the developer's machine):

| Hook | Fires | Typical use |
|---|---|---|
| `pre-commit` | Before a commit message editor opens, after staging | Linting, formatting, `shellcheck`, unit tests, secret scanning — fix or block *before* a bad commit is created |
| `commit-msg` | After the commit message is written, before the commit is finalized | Validate message format (Conventional Commits), reject and let the developer re-edit |
| `pre-push` | Before `git push` sends objects to the remote | Run a fuller test suite, block push of `WIP` commits |
| `post-checkout` / `post-merge` | After checkout/merge completes | Auto-install dependencies when `package.json` changed, warn about `.env` differences |

**Server-side hooks** (run on the Git server, can't be bypassed by a client):

| Hook | Fires | Typical use |
|---|---|---|
| `pre-receive` | Before any ref update is accepted | Hard enforcement — reject non-fast-forwards, enforce signed commits, run policy checks the client-side hook can't guarantee ran |
| `update` | Once per branch being updated | Per-branch policy (e.g., block direct pushes to `main`) |
| `post-receive` | After the push is accepted | Trigger CI, notify Slack, deploy |

**Why the client/server split matters:** a client-side hook is a courtesy — anyone can skip it with `git commit --no-verify` or by simply not installing it. A server-side hook (or the GitHub/GitLab equivalent — required status checks, branch protection) is the actual enforcement boundary, because the client has no way to bypass logic that runs on infrastructure it doesn't control. In practice: use client-side hooks (`pre-commit`, `husky`) for **fast feedback** so developers catch problems in seconds instead of waiting for CI, and use CI + branch protection as the **real gate**, since it can't be skipped with a flag.

## `pre-commit` — the Python framework

`pre-commit` (pre-commit.com) is a language-agnostic framework for managing and running hook scripts, distributed and versioned via a config file rather than hand-copied into `.git/hooks/`.

Install and wire it up:
```bash
pip install pre-commit          # or: brew install pre-commit / pipx install pre-commit
pre-commit install              # writes .git/hooks/pre-commit -> delegates to the framework
```

Configuration lives in `.pre-commit-config.yaml` at the repo root, committed like any other file — this is what makes hooks **shared across the team** instead of living only in one developer's local `.git/hooks/`:

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files

  - repo: https://github.com/koalaman/shellcheck-precommit
    rev: v0.10.0
    hooks:
      - id: shellcheck
        args: ["--severity=warning"]
```

Each `repo:` entry points at a separate Git repository that defines hooks (via its own `.pre-commit-hooks.yaml`); `rev:` pins an exact tag/commit so the hook's behavior doesn't silently change across machines or CI runs — this is a supply-chain control, not just a version pin: an unpinned or floating `rev` means a compromised upstream hook repo could inject arbitrary code that runs on every developer's commit. `pre-commit` clones each `repo:` once into `~/.cache/pre-commit`, builds an isolated environment for it (its own virtualenv for Python-based hooks, for example), and reuses that cache across repos and runs — so the first `pre-commit run` is slow (environment bootstrap) and subsequent ones are fast.

Running it:
```bash
pre-commit run                    # runs configured hooks against staged files only
pre-commit run --all-files        # runs against the entire repo (do this once after adding a new hook)
pre-commit autoupdate             # bumps every `rev:` to the latest tag
```

If a hook modifies files (like `trailing-whitespace` or a formatter), the commit is aborted so you can review the changes and re-stage — `pre-commit` never silently commits auto-fixed content.

## `husky` — the same idea for Node projects

`husky` manages Git hooks for JavaScript/Node repos, wiring `package.json` scripts into actual hook files instead of a YAML manifest.

```bash
npm install --save-dev husky
npx husky init                       # creates .husky/ dir, adds "prepare": "husky" to package.json
echo "npx --no-install commitlint --edit \"\$1\"" > .husky/commit-msg
chmod +x .husky/commit-msg
```

The `"prepare"` script in `package.json` is the key mechanism: `npm install` runs `prepare` automatically for every contributor and in CI, which re-installs the hooks — so, like `pre-commit install`, nobody has to remember a manual setup step. Hook files under `.husky/` are plain shell scripts committed to the repo, so (again, like `.pre-commit-config.yaml`) the hook definitions travel with the codebase instead of living only in one developer's `.git/hooks/`.

## Conventional Commits

Conventional Commits (conventionalcommits.org) is a lightweight, machine-parseable specification for commit message structure:

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

```
feat(auth): add refresh-token rotation
fix(api): handle null pointer on empty payload
chore(deps): bump express to 4.19.2
docs(readme): clarify local setup steps

feat(payments): support multi-currency checkout

BREAKING CHANGE: `charge()` now requires a `currency` argument
```

Common `type` values: `feat` (new feature), `fix` (bug fix), `chore` (tooling/maintenance, no production code change), `docs`, `refactor`, `test`, `perf`, `ci`, `build`. `scope` is an optional parenthesized noun narrowing where the change applies (`auth`, `api`, `deps`). A `BREAKING CHANGE:` footer (or a `!` right after the type/scope, e.g. `feat(api)!:`) flags an incompatible API change regardless of `type`.

**Why this format specifically, and not just "write good commit messages":** the format is *parseable*, which is what turns commit history into an input for automation rather than only prose for humans:
- **Automated semantic versioning** — tools like `semantic-release` or `standard-version` scan commits since the last tag: any `fix` bumps the patch version, any `feat` bumps minor, any `BREAKING CHANGE` bumps major (following SemVer's own rules exactly). No human decides the version number.
- **Automated changelog generation** — the same tools group commits by `type` into a generated `CHANGELOG.md` (a "Features" section from `feat` commits, a "Bug Fixes" section from `fix` commits) with zero manual changelog editing.
- **Signal in code review and history** — `git log --oneline` becomes scannable; `feat` vs `chore` vs `fix` tells a reviewer or future archaeologist what *kind* of change they're looking at before reading the diff.

## `commitlint` — enforcing the spec

`commitlint` parses a commit message against a configured rule set and rejects it if it doesn't conform. It's typically wired into the `commit-msg` hook (via `husky`, above) because that's the point in the lifecycle where the *message itself* is available to validate.

```bash
npm install --save-dev @commitlint/cli @commitlint/config-conventional
```

`commitlint.config.js`:
```js
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'header-max-length': [2, 'always', 72],
    'body-max-line-length': [2, 'always', 100],
    'scope-case': [2, 'always', 'lower-case'],
  },
};
```

Wired via the `.husky/commit-msg` hook shown above (`npx --no-install commitlint --edit "$1"` — `$1` is the path Git passes to `commit-msg` hooks containing the message being validated). A non-conforming commit message causes the hook to exit non-zero, which aborts the commit before it's created — the developer edits and retries, rather than a bad message ever landing in history. `pre-commit` can enforce the same spec without Node at all via the `commitizen`/`conventional-pre-commit` hook, using the `commit-msg` stage:

```yaml
  - repo: https://github.com/compilerla/conventional-pre-commit
    rev: v3.4.0
    hooks:
      - id: conventional-pre-commit
        stages: [commit-msg]
```

## Points to Remember

- Hooks are local scripts in `.git/hooks/`, never committed by default — `pre-commit` and `husky` exist specifically to make hook *definitions* shareable/versioned via a committed config file (`.pre-commit-config.yaml`, `.husky/*`).
- Client-side hooks (`pre-commit`, `commit-msg`, `pre-push`) are fast feedback and can be bypassed (`--no-verify`); server-side hooks / CI + branch protection are the real, unbypassable enforcement boundary.
- `pre-commit`'s `rev:` pin on each hook repo is a supply-chain safety control, not just version stability — an unpinned hook source is an arbitrary-code-execution risk on every commit.
- Conventional Commits' `type(scope): description` format is machine-parseable specifically so tooling (`semantic-release`, `commitlint`) can derive version bumps and changelogs from history automatically.
- `commitlint` validates the message in the `commit-msg` stage (message already exists by then); `shellcheck`/formatters validate file content in the `pre-commit` stage (before the commit object exists) — different hooks fire at different points because they operate on different things.

## Common Mistakes

- Writing a `pre-commit` hook by hand in `.git/hooks/` and assuming teammates get it after they `git pull` — hooks aren't tracked by Git; without `pre-commit`/`husky` (or a documented manual install step) it silently only runs for the author.
- Forgetting to run `pre-commit install` (or relying on `husky`'s `prepare` script actually firing) after cloning a repo, then being confused when commits with lint errors go through locally but fail in CI.
- Treating a client-side hook as sufficient enforcement — a teammate (or CI runner without hooks installed) can bypass it entirely with `--no-verify`; the real gate has to live in CI/branch protection.
- Floating `rev: main` or omitting `rev:` in `.pre-commit-config.yaml` instead of pinning a tag/SHA, which reintroduces the exact supply-chain risk pinning exists to prevent.
- Writing a commit message that says *what* changed but not *why*, even when it technically satisfies Conventional Commits syntax (`fix: update code`) — the spec disciplines structure, not content quality; `type(scope): description` still needs a real description.
