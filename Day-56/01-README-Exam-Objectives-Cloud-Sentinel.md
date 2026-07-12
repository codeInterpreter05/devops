# Day 56 — Terraform Associate Exam Prep: Exam Objectives, Terraform Cloud & Sentinel

**Phase:** 1 – Core DevOps | **Week:** W9 | **Domain:** Terraform | **Flag:** 📌

## Brief

The HashiCorp Terraform Associate (003) certification is one of the most recognized IaC credentials in DevOps hiring — recruiters and hiring managers use it as a fast proxy for "this person has used Terraform for real, not just copy-pasted a tutorial." Passing it (and, more importantly, being able to *talk through* its material in an interview) requires knowing the exam's 9 objective domains cold, and understanding the layer above open-source Terraform that most self-taught engineers never touch: Terraform Cloud/Enterprise and policy-as-code with Sentinel. This note covers the exam shape and that governance layer; the next two notes go deep on state management and the CLI utilities the exam (and real debugging) leans on most.

This day is split into three focused files:

1. **This file** — the 9 exam objective domains, and Terraform Cloud/Enterprise + Sentinel.
2. **[02-README-State-Management.md](02-README-State-Management.md)** — `terraform state` subcommands in depth.
3. **[03-README-CLI-Utilities.md](03-README-CLI-Utilities.md)** — `taint`, `import`, `graph`, `fmt`, `validate`.

## The 9 exam objective domains

HashiCorp publishes the exact objective list — know it as a checklist, not vague vibes:

1. **Understand Infrastructure as Code (IaC) concepts** — why declarative beats imperative for infra, idempotency, drift.
2. **Understand Terraform's purpose (vs other IaC)** — where it sits relative to Ansible/Chef/CloudFormation/Pulumi: Terraform is declarative, provider-agnostic (multi-cloud via providers), and manages a state file as its own record of truth — configuration management tools like Ansible are typically imperative/procedural and don't maintain equivalent persistent state.
3. **Understand Terraform basics** — providers, resources, data sources, variables, outputs, the core CLI workflow (`init`, `plan`, `apply`, `destroy`).
4. **Use the Terraform CLI (outside of core workflow)** — `fmt`, `validate`, `taint` (deprecated in favor of `-replace`), `import`, `workspace`, `graph`, `output`, `console`.
5. **Interact with Terraform modules** — module sourcing (local path, Registry, Git), input/output variables, versioning constraints, the public Terraform Registry.
6. **Navigate Terraform workflow** — `init` → `plan` → `apply` → `destroy`, and what each phase actually does under the hood.
7. **Implement and maintain state** — local vs. remote backends, locking, sensitive data in state, `terraform_remote_state` data source, state manipulation commands.
8. **Read, generate, and modify configuration** — HCL syntax, meta-arguments (`count`, `for_each`, `depends_on`, `lifecycle`), functions, expressions, dynamic blocks.
9. **Understand Terraform Cloud and Enterprise capabilities** — remote runs, Sentinel policy-as-code, private registry, workspaces-as-a-service, VCS-driven runs.

Domain 9 is the one most self-taught engineers skip because they've only used open-source Terraform CLI + a self-managed S3 backend — but the exam weights it meaningfully, and understanding it is genuinely useful once you're at a company big enough to need governance over who can apply what.

## Terraform Cloud vs. Terraform Enterprise vs. plain OSS

| | Terraform OSS (CLI) | Terraform Cloud (TFC) | Terraform Enterprise (TFE) |
|---|---|---|---|
| Where it runs | Your laptop/CI runner | HashiCorp-hosted SaaS | Self-hosted (your infra) |
| State storage | Local file or self-managed backend (S3+DynamoDB, etc.) | Managed remote state, built-in locking | Same as TFC, on-prem |
| Remote execution | No — plan/apply run wherever you invoke `terraform` | Yes — runs execute on HashiCorp's infrastructure | Yes — runs execute on your infrastructure |
| Policy as code | No | Sentinel (paid tiers) / OPA | Sentinel / OPA |
| Access control | None built-in (whatever your backend/CI provides) | Teams, RBAC, SSO (paid tiers) | Same, self-hosted |
| VCS integration | Manual (`git clone` + `terraform apply`) | Native — push-to-plan on PR, plan output as PR comment | Same, self-hosted |

The practical reason organizations move from OSS to TFC/TFE: **remote state + locking + a consistent execution environment removes "it applied differently from my laptop than from CI" class of bugs**, and centralizes *who* can approve an `apply` against production.

## Sentinel — policy as code

Sentinel is HashiCorp's policy-as-code framework, evaluated **between `plan` and `apply`** in Terraform Cloud/Enterprise (paid tiers) — it inspects the plan's proposed changes and can block an apply that violates an organizational rule, before anything is actually created or destroyed.

```
# example.sentinel — deny any S3 bucket without an environment tag
import "tfplan/v2" as tfplan

s3_buckets = filter tfplan.resource_changes as _, rc {
    rc.type is "aws_s3_bucket" and
    (rc.change.actions contains "create" or rc.change.actions contains "update")
}

main = rule {
    all s3_buckets as _, bucket {
        bucket.change.after.tags contains "environment"
    }
}
```

Sentinel policies have three enforcement levels: **advisory** (logs a warning, never blocks), **soft-mandatory** (blocks, but an authorized user can override), and **hard-mandatory** (blocks unconditionally, no override). This tiering matters operationally — you roll out a new policy as advisory first to see what it *would* have blocked across real runs before making it hard-mandatory and breaking people's workflows unexpectedly.

**Why this belongs in the exam and in interviews:** it's the answer to "how do you stop an engineer from `terraform apply`-ing a public S3 bucket or an oversized, budget-busting instance type" — a real governance problem every platform team eventually hits, distinct from just "code review the Terraform diff," because Sentinel evaluates the actual computed *plan*, not just the HCL source.

## Points to Remember

- Know all 9 exam domains as a literal checklist — domain 9 (TFC/TFE/Sentinel) is the one self-taught engineers most often under-prepare for.
- TFC/TFE add: remote state with locking, remote execution, VCS-driven runs, RBAC/SSO, and Sentinel/OPA policy-as-code — none of which plain OSS Terraform provides on its own.
- Sentinel evaluates the **plan** (the concrete, computed set of changes) between `plan` and `apply` — not the raw HCL — which is why it can catch things static code review can't (e.g., a `for_each` that expands into 50 non-compliant resources).
- Sentinel enforcement levels: advisory (warn only) → soft-mandatory (override possible) → hard-mandatory (no override) — roll out new policies advisory-first.
- Terraform Enterprise = Terraform Cloud's feature set, self-hosted — same capabilities, different deployment model (useful for orgs with data-residency/air-gap requirements).

## Common Mistakes

- Treating "I use Terraform" and "I've used Terraform Cloud/Sentinel" as interchangeable in an interview — interviewers at platform-team-mature companies will probe domain 9 specifically and it's an easy tell if you've never touched remote runs or policy-as-code.
- Assuming Sentinel policies check your HCL source code — they don't; they evaluate the rendered plan JSON, which is why writing a Sentinel test means understanding the `tfplan` import's structure, not your module's variable names.
- Rolling a new Sentinel policy straight to hard-mandatory without an advisory soak period, breaking every team's pipeline simultaneously on rollout day.
- Confusing Terraform Cloud's free tier (state storage + basic remote runs) with its paid-tier-only features (Sentinel, SSO, audit logging) and being surprised those aren't available on the free plan.
- Not knowing that OSS Terraform has **no built-in access control** at all — anyone with valid cloud credentials and backend access can `apply` anything; access control in OSS setups is entirely a function of how you've locked down CI/CD and the backend, not something Terraform itself enforces.
