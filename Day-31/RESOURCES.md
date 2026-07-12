# Day 31 — Resources: Terraform CI & Advanced Patterns

## Primary (assigned)

- **Atlantis documentation** (runatlantis.io) — the assigned starting point. Read "How it Works" and the GitHub Actions comparison to understand exactly what a purpose-built PR-automation tool buys you over hand-rolled CI.

## Deepen your understanding

- **Gruntwork — Terragrunt documentation** (terragrunt.gruntwork.io) — the official docs, particularly the "Keep your Terraform code DRY" and "Execute Terraform commands on multiple modules at once" pages, covering exactly the `include`/`dependency`/`run-all` patterns from today.
- **HashiCorp — "Automate Terraform with GitHub Actions" tutorial** (developer.hashicorp.com/terraform/tutorials/automation) — official walkthrough building the plan-on-PR/apply-on-merge pipeline from scratch.
- **Checkov documentation** (checkov.io) — the full built-in policy catalog searchable by cloud provider/resource type; useful for understanding what's covered before writing custom Sentinel/OPA rules for anything not already built in.
- **Open Policy Agent — "Terraform" tutorial** (openpolicyagent.org) — official OPA docs specifically on evaluating Terraform plan JSON with Rego and `conftest`.

## Reference / lookup

- **Infracost documentation** (infracost.io/docs) — CLI reference and CI integration guides (GitHub Actions, GitLab CI, Atlantis, Terraform Cloud run tasks).
- **AWS — "Configuring OpenID Connect in Amazon Web Services"** (docs.github.com, GitHub's official guide) — the exact steps for the OIDC setup in Lab 2.

## Practice

- **Gruntwork's `terragrunt-infrastructure-live-example` repo** (GitHub) — a real, browsable example of a multi-environment Terragrunt live repository structured the way production setups actually look.
