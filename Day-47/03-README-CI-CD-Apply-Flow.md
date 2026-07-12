# Day 47 — Terraform on AWS: CI/CD Apply Flow (Atlantis / GitHub Actions) & Outputs

**Phase:** 1 – Core DevOps | **Week:** W7 | **Domain:** Terraform | **Flag:** 📌 Reference

## Brief

Remote state (previous file) solves *where* Terraform's source of truth lives; this file solves *who is allowed to change it, and how*. Running `terraform apply` from a laptop is the single biggest governance gap in most early-stage infrastructure setups — no audit trail beyond git history, no consistent plan-before-apply enforcement, no PR-based review of the actual planned changes. Atlantis and GitHub Actions are the two dominant answers to "how do we make Terraform changes go through the same review process as application code."

## The GitOps-for-infrastructure principle

The pattern both tools implement: **the plan is generated automatically and posted where humans review it (a PR), and apply only happens after both human approval and a merge** — turning infrastructure changes into the same reviewed, auditable process as application code changes, instead of a special out-of-band process only a few people can run correctly.

## Atlantis

Atlantis is a **self-hosted** (or Atlantis Cloud-hosted) service that listens for pull request webhooks and runs Terraform commands in response to PR comments, purpose-built for this exact workflow.

**Mechanics:**
1. A PR is opened modifying `.tf` files in a directory Atlantis is configured to watch.
2. Atlantis automatically runs `terraform plan` in that directory and **posts the plan output as a PR comment** — no one needs to run anything locally to see the diff.
3. A reviewer inspects the plan in the PR comment, exactly like reviewing a code diff.
4. Once approved (and Atlantis can enforce this — requiring an approval before allowing apply), someone comments `atlantis apply` on the PR, which triggers `terraform apply` and posts the result back as another comment.
5. Merging the PR is typically configured to happen *after* a successful apply (or Atlantis can be configured to auto-merge on successful apply) — keeping git history and actual infrastructure state in sync.

**`atlantis.yaml` repo config** defines which directories are Terraform projects, what workflow (custom plan/apply commands, e.g., adding `tflint`/`checkov` steps) applies to each, and locking behavior — Atlantis natively **locks a directory/workspace combination** while a PR has an open, un-applied plan against it, preventing two PRs from racing to apply conflicting changes to the same state (a second layer of locking on top of the backend's own state lock from file 02).

**Why teams choose Atlantis specifically**: it's Terraform-native and self-hosted (full control, no dependency on a specific CI vendor's Terraform-specific tooling), and its PR-comment-driven workflow (`atlantis plan`, `atlantis apply`, `atlantis unlock`) is purpose-built for iterative infrastructure review — re-planning after a PR update happens automatically, which a generic CI pipeline needs to be explicitly wired to replicate.

## GitHub Actions apply flow

The alternative — using GitHub Actions directly with the Terraform CLI (or HashiCorp's official `hashicorp/setup-terraform` action) — trades some of Atlantis's Terraform-specific conveniences for using infrastructure teams already have (no separate service to host/maintain) and tighter integration with existing GitHub-native approval/environment-protection features.

```yaml
name: terraform
on:
  pull_request:
    paths: ["environments/prod/**"]
  push:
    branches: [main]
    paths: ["environments/prod/**"]

jobs:
  plan:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - run: terraform -chdir=environments/prod init
      - run: terraform -chdir=environments/prod plan -out=tfplan
      - uses: actions/github-script@v7   # post plan as PR comment
        with: { script: "// post plan output to PR" }

  apply:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production   # GitHub Environment: requires manual approval before this job runs
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - run: terraform -chdir=environments/prod init
      - run: terraform -chdir=environments/prod apply -auto-approve
```

**The key mechanism enforcing review here is a GitHub `environment` with required reviewers** — the `apply` job is gated behind a manual approval step configured on the `production` environment, functionally equivalent to Atlantis's "comment `atlantis apply`" gate, just implemented with GitHub-native primitives instead of a separate service.

**Atlantis vs. GitHub Actions — the honest tradeoff** (this comes up directly if asked to justify a choice in an interview):
- Atlantis: purpose-built UX for Terraform review (rich plan comments, automatic re-plan on push, native per-directory locking), but it's another service to host, secure, and keep patched — real operational overhead.
- GitHub Actions: no separate service, uses infrastructure/permissions you already manage, but you're hand-rolling plan-commenting, per-directory locking, and drift detection that Atlantis gives you natively — more initial engineering work for equivalent safety guarantees.
- Team size/maturity is usually the deciding factor: a platform team maintaining many repositories and wanting a consistent, low-friction review experience across all of them often justifies running Atlantis; a smaller team with one or two Terraform repos often finds plain GitHub Actions perfectly sufficient without adding another service to operate.

## Terraform outputs — surfacing what downstream needs

```hcl
output "cluster_endpoint" {
  description = "EKS control plane endpoint"
  value       = module.eks.cluster_endpoint
}

output "cluster_certificate_authority_data" {
  value     = module.eks.cluster_certificate_authority_data
  sensitive = true
}

output "oidc_provider_arn" {
  value = module.eks.oidc_provider_arn
}
```

Outputs matter for three practical reasons beyond just "printing something at the end of `apply`":
1. **Cross-state referencing** — the exact values `terraform_remote_state` (file 02) reads from another component's state.
2. **CI/CD pipeline consumption** — a downstream CD step (e.g., a Helm deploy job) reading `terraform output -json` to get the cluster name/endpoint it needs to `aws eks update-kubeconfig` against, without hardcoding values that could drift from reality.
3. **`sensitive = true`** suppresses a value from CLI/plan output and CI logs (it's still stored in plaintext in the state file itself, which is exactly why state file encryption and access control from file 02 matters) — mark anything secret-shaped this way, but don't treat it as a substitute for actually keeping the state file access-controlled.

## Points to Remember

- Both Atlantis and GitHub Actions implement the same principle: plan is automatic and visible in a PR, apply requires explicit human approval and happens only after merge (or is gated to run only on `main`).
- Atlantis adds Terraform-specific conveniences (rich plan comments, automatic re-plan, native per-directory locking) at the cost of hosting/maintaining another service; GitHub Actions reuses existing infrastructure at the cost of more manual wiring for equivalent guarantees.
- A GitHub `environment` with required reviewers is the GitHub-native mechanism for gating `apply` behind approval — the direct equivalent of Atlantis's comment-triggered apply gate.
- `terraform output` is how downstream automation (CD steps, other Terraform state files) consumes values from a run without hardcoding them.
- `sensitive = true` on an output hides it from CLI/log output but does not encrypt it within the state file itself — state file security is still the real control.

## Common Mistakes

- Running `terraform apply` manually from a laptop against shared/production infrastructure "just this once," bypassing the entire PR-review/plan-visibility process the team otherwise relies on.
- Configuring Atlantis or a GitHub Actions workflow to auto-apply on every PR without a human-approval gate, effectively giving anyone who can open a PR (or worse, anyone who can push to certain branches) the ability to change production infrastructure unreviewed.
- Forgetting `paths:` filters in a GitHub Actions trigger, causing the Terraform plan/apply workflow to run (and potentially lock state) on every unrelated PR to the repository, not just ones touching Terraform files.
- Treating `sensitive = true` as sufficient protection for a secret output, forgetting the underlying value still sits in plaintext in the state file — real protection requires state file encryption and tightly scoped access, not just CLI-output suppression.
- Not wiring automatic re-plan on PR updates (native in Atlantis, something you must build yourself in a plain GitHub Actions setup) — reviewing a stale plan comment against a PR that has since been updated is a real, easy-to-miss source of applying something different from what was actually reviewed.
