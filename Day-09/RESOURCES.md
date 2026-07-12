# Day 9 — Resources: Git Deep Dive II — Workflows

## Primary (assigned)

- **Trunk-Based Development website** (trunkbaseddevelopment.com) — the assigned starting point for this day. The definitive reference for the trunk-based model: short-lived branches, release patterns, and how it compares to GitFlow, written by practitioners who've run it at scale.

## Deepen your understanding

- **A successful Git branching model** by Vincent Driessen (nvie.com/posts/a-successful-git-branching-model) — the original GitFlow post. Worth reading even if you end up not using GitFlow, since it's the model most branching discussions still get compared against.
- **Conventional Commits specification** (conventionalcommits.org) — the actual spec text: full grammar, examples, and the rationale for why the format is machine-parseable.
- **pre-commit** (pre-commit.com) — official docs for the framework: supported hook repos, configuration reference, and how the caching/environment-isolation model works under the hood.
- **Pro Git — Customizing Git: Git Hooks** (git-scm.com/book/en/v2/Customizing-Git-Git-Hooks) — the free canonical book chapter covering every client-side and server-side hook Git supports, straight from the source.

## Reference

- **GitHub Docs — About protected branches** (docs.github.com, search "About protected branches") — the authoritative list of what each branch protection setting does, including required status checks, linear history, and required signatures.
- **GitHub Docs — Signing commits** and **GitLab Docs — Signed commits** — platform-specific walkthroughs for both GPG and SSH-based commit signing, and exactly how each platform computes the "Verified" badge.
- **commitlint** (commitlint.js.org) — configuration reference for rule sets (`@commitlint/config-conventional`) and how to write custom rules.
- **husky** (typicode.github.io/husky) — official docs for Node-side hook management, including the `prepare` script mechanism that auto-installs hooks for every contributor.

## Practice

- **ShellCheck's own wiki** (github.com/koalaman/shellcheck/wiki) — every warning code (`SC2086`, etc.) explained with a bad/good example pair; useful for understanding *why* a shellcheck finding matters, not just that it fired.
- Fork any small personal shell-script or Node repo and retrofit it with `.pre-commit-config.yaml` + `commitlint`/`husky` exactly as in today's lab — doing it against real, pre-existing (imperfect) code surfaces edge cases a fresh scratch repo won't.
