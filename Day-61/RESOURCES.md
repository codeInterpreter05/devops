# Day 61 — Resources: GitHub Actions Fundamentals

## Primary (assigned)

- **GitHub Actions documentation** (docs.github.com/actions) — free, the assigned starting point. Start with "Understanding GitHub Actions" then "Workflow syntax for GitHub Actions" for the full `on`/`jobs`/`steps` reference.

## Deepen your understanding

- **GitHub Actions "Security hardening for GitHub Actions"** (docs.github.com/actions/security-guides) — the canonical reference on the script-injection risk covered in file 2 of today's notes (untrusted `github.event.*` interpolation).
- **`act` GitHub repo** (github.com/nektos/act) — README doubles as a practical guide to running workflows locally, useful for understanding what actually differs between local emulation and real GitHub runners.
- **GitHub Actions Marketplace** (github.com/marketplace?type=actions) — browse widely-used actions (`actions/checkout`, `actions/cache`, `docker/build-push-action`) and read their own READMEs for the full `with:` input reference beyond what's in this note.

## Reference / lookup

- **Contexts and expression syntax reference** (docs.github.com/actions/learn-github-actions/contexts) — the exhaustive list of every context and field (`github.*`, `runner.*`, `job.*`, etc.).
- **`docker/build-push-action` docs** — the authoritative source for `cache-from`/`cache-to` syntax and other GitHub Actions cache backend options (`type=gha`, `type=registry`, `type=local`).

## Practice

- **GitHub Skills — "Hello GitHub Actions"** (skills.github.com) — free, interactive, GitHub's own hands-on course that walks through building workflows directly in a sandboxed repo.
- Fork any small open-source repo and add a matrix-based CI workflow yourself, deliberately breaking `fail-fast` behavior once to see the cancel-cascade in the Actions UI — the fastest way to internalize the difference between `true`/`false`.
