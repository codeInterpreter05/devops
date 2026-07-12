# Day 69 — Jenkins: Migration Path to GitHub Actions

**Phase:** 2 – CI/CD & Security | **Week:** W11 | **Domain:** CI/CD

## Brief

"Your company uses Jenkins and wants to migrate to GitHub Actions — what's your strategy?" is one of the most common CI/CD interview questions for anyone with Jenkins on their résumé, precisely because it tests something beyond tool knowledge: can you plan a *safe, incremental* migration of business-critical infrastructure without a risky big-bang cutover? This file covers the conceptual mapping between the two systems and a realistic phased migration plan.

## Why teams migrate away from Jenkins at all

- **Operational burden**: Jenkins is self-hosted infrastructure — you patch it, scale its agents, manage plugin compatibility (a notoriously fragile ecosystem where upgrading Jenkins core can break several plugins simultaneously), and are responsible for its uptime. GitHub Actions (and other SaaS CI) offload all of that.
- **Native integration**: GitHub Actions triggers directly off GitHub events (PR opened, label added, release published) with zero webhook plumbing; Jenkins needs the GitHub plugin, webhook configuration, and credential wiring to achieve the same.
- **Ecosystem/marketplace**: GitHub Actions' Marketplace has a large library of pre-built actions maintained by vendors themselves (AWS, Docker, Slack) versus Jenkins plugins of wildly varying quality and maintenance status.
- **Cost model difference**: Jenkins' "cost" is the infrastructure and operational time to run it yourself; GitHub Actions bills per-minute for hosted runners (or is free-ish if you bring your own self-hosted runners) — the tradeoff is operational effort vs. metered usage cost, not a simple "one is cheaper" answer.

## Conceptual mapping

| Jenkins concept | GitHub Actions equivalent |
|---|---|
| `Jenkinsfile` | `.github/workflows/*.yml` |
| `pipeline { stages { stage(...) } }` | `jobs: <job>: steps: [...]` |
| `agent` / static or Kubernetes agent | `runs-on: ubuntu-latest` (hosted) or self-hosted runner |
| Shared Library (`vars/*.groovy`) | Reusable/composite Action, or a reusable workflow (`workflow_call`) |
| `credentials('id')` | Encrypted repo/org **Secrets**, or OIDC (no stored secret at all) |
| Plugins (Slack notify, JUnit publish, etc.) | Marketplace Actions (`slackapi/slack-github-action`, `actions/upload-artifact` + built-in test reporting) |
| Multibranch Pipeline job | Just works natively — every branch/PR triggers workflows automatically |
| Blue Ocean visualization | Native Actions UI (built-in, no plugin needed) |

A minimal side-by-side for the same "build, scan, deploy on main" pipeline:

```groovy
// Jenkinsfile
pipeline {
    agent any
    stages {
        stage('Build')  { steps { sh 'docker build -t myapp .' } }
        stage('Scan')   { steps { sh 'trivy image --exit-code 1 --severity CRITICAL myapp' } }
        stage('Deploy') { when { branch 'main' } steps { sh './deploy.sh' } }
    }
}
```

```yaml
# .github/workflows/build.yml
name: build
on: [push, pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t myapp .
      - name: Scan
        run: docker run aquasec/trivy image --exit-code 1 --severity CRITICAL myapp
      - name: Deploy
        if: github.ref == 'refs/heads/main'
        run: ./deploy.sh
```

## A realistic phased migration plan

**Phase 1 — Inventory and triage.** List every Jenkins job/Jenkinsfile in use, and classify each by risk/complexity: simple build-test-deploy pipelines vs. ones with heavy shared-library dependencies, exotic plugins, or manual approval gates. Migrate low-risk, high-repetition pipelines first (they validate the new tooling cheaply) — don't start with the scariest, most business-critical pipeline.

**Phase 2 — Run in parallel, don't cut over immediately.** For each migrated pipeline, run the GitHub Actions workflow *alongside* the existing Jenkins job for a defined period (a few weeks is typical), comparing outputs/timings, without yet making the new pipeline the source of truth for deploys. This catches subtle behavior differences (different default shell, different working directory assumptions, different secret injection behavior) before anything depends on the new pipeline being correct.

**Phase 3 — Migrate shared logic first, not each pipeline independently.** Just as Jenkins shared libraries centralized common logic, convert that shared logic into GitHub Actions **reusable workflows** (`workflow_call`) or **composite actions** *before* migrating the many individual app pipelines that will consume it — otherwise you re-derive the same "build+scan+deploy" boilerplate 50 separate times in YAML instead of once in Groovy.

**Phase 4 — Cut over incrementally, team by team or repo by repo**, keeping Jenkins running read-only/available for rollback until confidence is high, then decommission Jenkins infrastructure last — after the riskiest pipelines have proven stable on the new system, not before.

**Phase 5 — Secrets and credentials migration deserves its own explicit step**: don't just copy Jenkins credential values into GitHub Secrets verbatim. This is the ideal moment to move to **OIDC-based cloud authentication** (GitHub Actions minting short-lived AWS/GCP/Azure credentials via OIDC, the same mechanism from Day 67's keyless signing) instead of carrying forward long-lived static credentials that Jenkins jobs may have accumulated over years.

## Points to Remember

- Migration should be phased and run in parallel before cutover — never a big-bang replace of a business-critical pipeline in one step.
- Migrate shared/common logic (Jenkins shared libraries → GitHub Actions reusable workflows/composite actions) *before* migrating the many individual pipelines that depend on it, to avoid duplicating boilerplate across every migrated repo.
- Start with low-risk, high-repetition pipelines to validate the new tooling and build confidence before tackling the most complex/critical Jenkins jobs.
- Secrets migration is a chance to upgrade security posture, not just copy values over — prefer OIDC-based short-lived cloud credentials over carrying forward long-lived static secrets.
- Jenkins' operational burden (self-hosted, plugin fragility, agent scaling) is the main driver toward SaaS CI — but GitHub Actions trades that for a metered, per-minute cost model, which is a real tradeoff to name explicitly in an interview answer, not just "GitHub Actions is better."

## Common Mistakes

- Attempting a single big-bang cutover of all Jenkins pipelines on a fixed date — any subtle behavioral difference (shell defaults, working directory, secret masking behavior) surfaces simultaneously across every team with no easy fallback.
- Migrating each application's pipeline independently before migrating shared logic, resulting in the same "build+scan+deploy" YAML copy-pasted across dozens of repos — recreating exactly the maintenance problem shared libraries were built to solve, just in a new tool.
- Copying long-lived static cloud credentials from Jenkins credential store directly into GitHub Secrets without evaluating OIDC — missing the best opportunity to eliminate long-lived secrets from the pipeline entirely.
- Decommissioning Jenkins (or its agents) before the riskiest/most complex pipelines have been fully validated on the new system — leaving no rollback path if a critical, rarely-triggered pipeline (e.g., an annual compliance job) turns out to be broken.
- Assuming feature parity is 1:1 — some Jenkins plugin behaviors (complex custom UI approval steps, certain enterprise plugin integrations) don't have an exact GitHub Actions equivalent and need to be redesigned, not just "translated" line by line.
