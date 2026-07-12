# Day 47 — Terraform on AWS: Remote State with Locking

**Phase:** 1 – Core DevOps | **Week:** W7 | **Domain:** Terraform | **Flag:** 📌 Reference

## Brief

Terraform state is the single most operationally dangerous file in your entire infrastructure toolchain — it's the source of truth mapping your HCL to real, running AWS resources, and losing or corrupting it (or two people applying against it simultaneously) can mean anything from a confusing plan to Terraform trying to destroy and recreate a live production EKS cluster. Remote state with locking is the non-negotiable baseline for any team (more than one person, or one person plus CI) working against the same infrastructure.

## Why local state doesn't survive contact with a team

Local state (`terraform.tfstate` sitting in your working directory) has three fatal problems the moment more than one person or process touches the same infrastructure:
1. **No sharing** — the next person to run `terraform apply` doesn't have your local state file, so Terraform thinks the resources you created don't exist and tries to create them again (or errors on name collisions).
2. **No locking** — two people running `terraform apply` at the same moment against the same local file can corrupt it or produce conflicting changes with no coordination.
3. **No history/durability** — a single accidentally-deleted or corrupted local file means Terraform loses all knowledge of what it's managing, with no way to recover short of `terraform import`-ing every resource back by hand.

## Remote state with S3 + DynamoDB (the classic AWS-native pattern)

```hcl
terraform {
  backend "s3" {
    bucket         = "my-org-terraform-state"
    key            = "prod/eks-cluster/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

- **S3 bucket** stores the actual state file — must have **versioning enabled** (so a bad `apply` that corrupts state can be rolled back to a prior version) and **encryption at rest** (state files contain resource attributes that can include sensitive values — database passwords set via a resource argument, for instance — so treat the bucket as sensitive data, not just config).
- **`key`** is the path within the bucket — the convention `<environment>/<component>/terraform.tfstate` is what enables one bucket to hold many isolated state files (see below).
- **DynamoDB table** (`terraform-locks`, needs just a single primary key `LockID` of type String) provides **locking**: before any `apply`/`plan` that would write state, Terraform writes a lock item to DynamoDB; a concurrent run attempting the same fails immediately with a clear "state is locked" error instead of racing. Terraform 1.6+ can alternatively use S3's native conditional-write locking without DynamoDB, but the S3+DynamoDB pattern remains extremely common and is what you'll see in most existing codebases and interview discussions.
- **Note**: as of newer Terraform/OpenTofu versions, `use_lockfile = true` on the S3 backend enables S3-native locking (via conditional writes) without needing a separate DynamoDB table — worth knowing this evolution exists, even though S3+DynamoDB is still the dominant pattern in production codebases today.

## State isolation — the design decision that actually matters

**One giant state file for everything is the single most common structural mistake** in real Terraform codebases, for two compounded reasons:
1. **Blast radius**: a mistake or a provider bug affecting *any* resource in that state file can affect `plan`/`apply` for *everything* in it — a bad plan on an unrelated S3 bucket can block or complicate an urgent EKS node group fix.
2. **Speed**: `terraform plan` refreshes (reads current status of) every resource in the state before showing a diff — a single state file with hundreds of resources across every environment makes every single `plan` slow, even for a one-line change.

**The standard mitigation: isolate state per environment and per logical component.**
```
prod/vpc/terraform.tfstate
prod/eks-cluster/terraform.tfstate
prod/rds/terraform.tfstate
staging/vpc/terraform.tfstate
staging/eks-cluster/terraform.tfstate
```
Each is a separate root module invocation with its own backend config (often just a different `key`), consuming outputs from other state files via `terraform_remote_state` data sources or (more modern, less coupling) via SSM Parameter Store / explicit variable passing rather than direct state-file cross-referencing:
```hcl
data "terraform_remote_state" "vpc" {
  backend = "s3"
  config = {
    bucket = "my-org-terraform-state"
    key    = "prod/vpc/terraform.tfstate"
    region = "us-east-1"
  }
}
# usage: data.terraform_remote_state.vpc.outputs.vpc_id
```

**Workspaces vs. separate state files/directories for environments** — a frequent point of confusion: Terraform **workspaces** (`terraform workspace new staging`) give you multiple state files under one backend configuration, switching the active one via a CLI command — convenient, but a real risk in practice: it's easy to run `apply` in the wrong workspace by mistake since the workspace is just ambient CLI state, not something visible in the code you're looking at. The more common and safer production pattern is **separate directories per environment** (`environments/prod/`, `environments/staging/`), each with its own explicit backend `key` — the environment is then unambiguous from which directory you're standing in, not from invisible session state.

## Points to Remember

- Local state fails immediately once more than one person/process touches the same infrastructure — no sharing, no locking, no durability guarantee.
- S3 (versioned, encrypted) + DynamoDB (lock table) is the classic AWS-native remote backend; newer Terraform/OpenTofu versions can achieve locking via S3 conditional writes alone (`use_lockfile`).
- One giant state file for everything is the most common real-world structural mistake — it maximizes blast radius and makes every `plan` slow regardless of the change's actual size.
- Isolate state per environment and per logical component (VPC, EKS, RDS as separate state files); cross-reference via `terraform_remote_state` data sources or explicit output passing (e.g., SSM Parameter Store).
- Prefer separate directories per environment over Terraform workspaces for anything production — workspaces are convenient but make "which environment am I about to apply to" dangerously invisible in the code itself.

## Common Mistakes

- Committing `terraform.tfstate` to git (instead of configuring a remote backend) — leaks sensitive resource attributes into version control history and gives you zero locking, a genuinely dangerous default for anything beyond a personal sandbox.
- Forgetting to enable versioning on the S3 state bucket, so a state-corrupting mistake (bad manual edit, botched `terraform state rm`) has no prior version to roll back to.
- Using one enormous state file across all environments and components "to keep things simple," then discovering that every `plan` takes minutes and a single unrelated resource error blocks an urgent unrelated fix.
- Switching Terraform workspaces absentmindedly (`terraform workspace select prod` from muscle memory) and running `apply` against the wrong environment because the active workspace wasn't visually obvious in the terminal at that moment.
- Cross-referencing another component's resources by hardcoding IDs/ARNs pulled from the console instead of via `terraform_remote_state` or a proper output-passing mechanism — breaks the moment the referenced resource is recreated with a new ID.
