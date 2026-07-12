# Day 76 — Resources: CI/CD & DevSecOps Review

## Primary (assigned)

- **Awesome GitHub Actions** (github.com/sdras/awesome-actions) — the assigned starting point. A curated list of Actions and example workflows; browse real, popular repos' `.github/workflows/` directories linked from or alongside it to find a pipeline worth auditing for today's hands-on activity.

## Deepen your understanding

- **"Keeping your GitHub Actions and workflows secure" (GitHub Security Lab blog series)** — the authoritative writeup of `pull_request_target` risks, script injection via untrusted `github.event.*` context values, and Action pinning guidance, written by the team that finds these bugs in the wild.
- **StepSecurity "Harden-Runner" docs and blog** (stepsecurity.io) — practical, example-driven material on GitHub Actions supply-chain hardening (egress monitoring, SHA pinning enforcement) that pairs directly with the audit checklist in file 2.
- **"Make It Stick: The Science of Successful Learning" (Brown, Roediger, McDaniel)** — not DevOps-specific, but the research foundation behind why closed-book recall/retrieval practice (this day's review method) outperforms passive rereading; useful if you want the cognitive-science "why" behind today's method.

## Reference

- **GitHub Actions security hardening guide** (docs.github.com/actions/security-guides): official reference for `permissions:`, OIDC configuration, and secret-handling best practices to check pipelines against.
- **Dependabot / Renovate docs**: both can automatically open PRs bumping pinned Action SHAs, directly addressing the maintenance trade-off of SHA-pinning recommendations.

## Practice

- **GitHub's public repo search** (`is:public path:.github/workflows`) — find real, active workflows to audit beyond whatever one repo you start with; auditing 3-4 different pipelines surfaces how common (or rare) each issue category actually is in practice.
- **Your own Day 73 capstone pipeline** — turn the audit checklist on your own repo from three days ago; auditing your own past work with fresh eyes is often more revealing than auditing a stranger's.
