# Day 31 — Terraform CI & Advanced Patterns: Terragrunt for DRY Code

**Phase:** 1 – Core DevOps | **Week:** W5 | **Domain:** Terraform | **Flag:** —

## Brief

Modules (Day 30) solve DRY *within* a single environment's resources. But once you have real dev/staging/prod environments, each with its own backend config, provider config, and variable values, you end up repeating the *scaffolding* around modules — the `terraform { backend {...} }` block, the `provider` block, the `remote_state` wiring — across every environment folder. **Terragrunt** is a thin wrapper around Terraform, from Gruntwork (the same team behind "Terraform: Up & Running"), built specifically to DRY up that remaining repetition. It's a very common real-world tool, and "have you used Terragrunt, and do you know when *not* to reach for it" is a legitimate interview question for any team running Terraform across multiple environments.

## The problem Terragrunt solves

Without Terragrunt, a typical multi-environment layout duplicates backend config in every environment:

```
live/
  dev/vpc/main.tf        # backend "s3" { bucket = "...", key = "dev/vpc/..." }
  staging/vpc/main.tf    # backend "s3" { bucket = "...", key = "staging/vpc/..." }
  prod/vpc/main.tf       # backend "s3" { bucket = "...", key = "prod/vpc/..." }
```

Each of those backend blocks is nearly identical except for the `key`. Multiply this by every module (vpc, ec2, rds, ...) across every environment, and you have dozens of near-duplicate boilerplate blocks that all need to change together if, say, the state bucket name changes.

## How Terragrunt DRYs this up

```
live/
  terragrunt.hcl          # root config: shared backend + provider generation
  dev/
    vpc/terragrunt.hcl     # points at the module, sets dev-specific inputs
  staging/
    vpc/terragrunt.hcl
  prod/
    vpc/terragrunt.hcl
```

```hcl
# live/terragrunt.hcl (root)
remote_state {
  backend = "s3"
  generate = { path = "backend.tf", if_exists = "overwrite" }
  config = {
    bucket         = "my-org-terraform-state"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = "ap-south-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}

generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite"
  contents  = <<-EOF
    provider "aws" {
      region = "ap-south-1"
    }
  EOF
}
```

```hcl
# live/dev/vpc/terragrunt.hcl
include "root" {
  path = find_in_parent_folders()
}

terraform {
  source = "git::https://github.com/org/tf-modules.git//vpc?ref=v1.2.0"
}

inputs = {
  cidr_block = "10.0.0.0/16"
  environment = "dev"
}
```

Every child `terragrunt.hcl` inherits the root's backend and provider generation via `include`, and the `key` is computed automatically from the file path (`path_relative_to_include()`) — so `dev/vpc` and `prod/vpc` automatically get distinct state paths without anyone typing them out. Running `terragrunt apply` inside `live/dev/vpc/` generates the actual `backend.tf`/`provider.tf` files on the fly, then shells out to the real `terraform` binary underneath.

## `run-all` — orchestrating many modules at once

```bash
cd live/dev
terragrunt run-all plan
terragrunt run-all apply
```

If `dev/vpc` and `dev/ec2` are separate Terragrunt units and `ec2` depends on `vpc`'s outputs (via a `dependency` block, Terragrunt's equivalent of `terraform_remote_state` but with a cleaner syntax), `run-all` topologically sorts them and applies in the correct order across the whole tree — something plain Terraform has no native concept of across separate root modules/state files.

```hcl
# live/dev/ec2/terragrunt.hcl
dependency "vpc" {
  config_path = "../vpc"
}

inputs = {
  vpc_id    = dependency.vpc.outputs.vpc_id
  subnet_id = dependency.vpc.outputs.subnet_ids[0]
}
```

## When Terragrunt is (and isn't) worth adopting

Terragrunt earns its keep once you have: multiple environments, multiple independent state files that need orchestrating together, and enough repeated backend/provider boilerplate that the DRY win is real. It's an **extra tool and extra layer of indirection** on top of Terraform — a new engineer now has to understand both Terraform *and* Terragrunt's generation/inheritance model to fully debug a run. For a single environment, or a small team already comfortable with plain modules + workspaces, the added complexity may not pay for itself. This "is the abstraction worth its cost" judgment call is exactly what interviewers are probing when they ask about it.

## Points to Remember

- Terragrunt DRYs up the *scaffolding* around Terraform (backend/provider config, cross-module orchestration) — it doesn't replace modules, which still DRY the actual resource definitions.
- `include` + `find_in_parent_folders()` is how child `terragrunt.hcl` files inherit shared root configuration without repeating it.
- `dependency` blocks are Terragrunt's cleaner alternative to `terraform_remote_state` for wiring one unit's outputs into another's inputs.
- `run-all` orchestrates plan/apply/destroy across an entire tree of Terragrunt units in dependency order — something plain Terraform has no built-in equivalent for across separate state files.
- Terragrunt is an extra abstraction layer with a real onboarding cost — adopt it because repeated boilerplate is a genuine, measured pain point, not by default.

## Common Mistakes

- Reaching for Terragrunt on day one of a small, single-environment project, adding indirection before there's enough real duplication to justify it.
- Forgetting that `generate` blocks overwrite files like `backend.tf`/`provider.tf` on every run — manually editing those generated files directly (instead of the `terragrunt.hcl` that produces them) leads to changes silently vanishing on the next run.
- Using `run-all apply` across an entire environment without first running `run-all plan` and reviewing it — the blast radius of a mistake is now "every module in this tree," not just one.
- Not pinning the module `source` `ref` in `terraform { source = ... }` blocks, hitting the same unpinned-module problem as Day 30 but now multiplied across every environment's Terragrunt unit.
- Assuming Terragrunt changes what Terraform actually does — it's a code generator and orchestration wrapper; the underlying `terraform plan`/`apply` semantics (state, locking, dependency graph) are unchanged.
