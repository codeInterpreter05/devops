# Day 31 — Terraform CI & Advanced Patterns: Terraform in CI & Drift Detection

**Phase:** 1 – Core DevOps | **Week:** W5 | **Domain:** Terraform | **Flag:** —

## Brief

Running `terraform apply` from someone's laptop against production is how infrastructure incidents start — no audit trail beyond "I think Priya ran it," no guarantee the applied plan matches what was reviewed, no consistent environment (their local Terraform version, their local AWS credentials). The fix is the same one application code went through years ago: **put infrastructure changes through a CI/CD pipeline** — plan automatically on every PR, require review of the plan output, apply only on merge. This is the single most concretely interview-relevant pattern in this entire domain, tied directly to today's assigned hands-on activity.

## The standard pattern: plan on PR, apply on merge

```
Engineer opens PR with .tf changes
        │
        ▼
CI runs: terraform fmt -check → terraform validate → terraform plan
        │
        ▼
Plan output posted as a PR comment (human-reviewable diff of infra changes)
        │
        ▼
Reviewer approves PR based on the code AND the plan output
        │
        ▼
PR merged to main
        │
        ▼
CI runs: terraform apply (using the merge commit, ideally the exact reviewed plan)
```

Why this shape specifically:
- **Plan on PR** turns an infrastructure change into something reviewable *before* it happens — a reviewer sees "this will destroy the prod RDS instance" in the PR the same way they'd see a risky code diff, instead of finding out after `apply` ran.
- **Apply only on merge to main** means there's exactly one place production changes originate from — a git commit, with an author, a PR link, and required approvals — never someone's local machine with ambient AWS credentials.
- **CI is the only thing with apply credentials** — engineers get PR review authority, not direct `apply` access, which is the actual answer to "how do you prevent engineers from running terraform apply locally against production" (this day's interview question).

## A working GitHub Actions implementation

```yaml
# .github/workflows/terraform.yml
name: Terraform
on:
  pull_request:
    paths: ["infra/**"]
  push:
    branches: [main]
    paths: ["infra/**"]

permissions:
  contents: read
  pull-requests: write   # needed to post the plan as a PR comment
  id-token: write         # needed for OIDC auth to AWS (no long-lived keys)

jobs:
  terraform:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: infra
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.7.5

      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/gha-terraform-role
          aws-region: ap-south-1

      - run: terraform fmt -check -recursive
      - run: terraform init
      - run: terraform validate

      - name: Plan
        if: github.event_name == 'pull_request'
        run: terraform plan -no-color -out=tfplan | tee plan_output.txt

      - name: Post plan as PR comment
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const plan = fs.readFileSync('infra/plan_output.txt', 'utf8').slice(-60000);
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `### Terraform Plan\n\`\`\`\n${plan}\n\`\`\``
            });

      - name: Apply
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: terraform apply -auto-approve tfplan
```

Note the use of **OIDC federation** (`aws-actions/configure-aws-credentials` with `role-to-assume`) instead of long-lived AWS access keys stored as GitHub secrets — GitHub Actions can present a short-lived, workflow-scoped identity token that AWS STS exchanges for temporary credentials, removing the risk of a leaked static key with standing access. `-auto-approve` is acceptable **only** in this specific CI context, because the human approval already happened via PR review — it is never acceptable for an interactive human session against prod (Day 29).

## Drift detection with a scheduled `terraform plan`

Infrastructure drifts even without anyone touching Terraform directly: a security team manually tightens a security group rule during an incident, an autoscaling event changes an ASG's desired count outside Terraform's awareness, or another tool modifies a tag. A scheduled CI job that runs `terraform plan` on a cron (with no `apply`) surfaces this before it causes confusion:

```yaml
on:
  schedule:
    - cron: "0 6 * * *"   # daily at 06:00 UTC

jobs:
  drift-check:
    steps:
      - run: terraform plan -detailed-exitcode
        # exit code 0 = no changes, 1 = error, 2 = changes detected (drift!)
      - name: Alert on drift
        if: steps.plan.outputs.exit-code == '2'
        run: <post to Slack/PagerDuty>
```

`-detailed-exitcode` is the mechanism that makes this automatable — plain `terraform plan` always exits 0 on success regardless of whether changes were found, so a naive CI check can't distinguish "no drift" from "drift found" without it.

## Atlantis — a purpose-built alternative to hand-rolled CI

**Atlantis** (self-hosted, open source, runatlantis.io — this day's assigned resource) is a dedicated Terraform-PR-automation server: it listens for PR webhook events, runs `plan` automatically, comments the output, and lets an authorized user trigger `apply` via a PR comment (`atlantis apply`) rather than a separate CI pipeline definition per repo. It's worth knowing as the "someone already built this so you don't have to hand-roll YAML" option, conceptually adjacent to what Terraform Cloud's VCS-driven runs provide (Day 30) but self-hosted and free.

## Points to Remember

- The core pattern is: plan (and post it for review) on every PR, apply only from CI after merge to main — never a human running `apply` locally against shared/production state.
- OIDC federation lets CI assume a short-lived, scoped AWS role instead of storing long-lived access keys as secrets — prefer it whenever the CI platform supports it.
- `terraform plan -detailed-exitcode` is what makes automated drift detection possible — it's the only way to distinguish "no changes" from "changes found" via exit code.
- Applying the *exact* plan reviewed in the PR (via a saved `-out` artifact carried between jobs) closes the gap between "what a human approved" and "what actually executed."
- Atlantis and Terraform Cloud both exist to replace hand-rolled CI YAML for this exact plan/apply/comment loop — worth evaluating before building it all from scratch.

## Common Mistakes

- Giving CI's AWS role (or, worse, individual engineers) standing `apply`-level permissions with no distinction between "can plan/review" and "can execute against prod."
- Running a fresh `terraform plan` at apply time instead of reusing the exact `-out` plan file that was reviewed — if infrastructure changed between PR approval and merge, the applied plan may not be the one anyone actually looked at.
- Storing long-lived AWS access keys as CI secrets when the CI platform supports OIDC federation — a leaked static key has no expiry; a leaked OIDC-derived temporary credential does.
- Not scoping CI triggers with `paths:` filters, causing every unrelated code change in a monorepo to trigger a full Terraform plan/apply cycle.
- Skipping scheduled drift detection entirely and only discovering manual/out-of-band infrastructure changes weeks later when an unrelated `apply` unexpectedly wants to "fix" something nobody remembers changing.
