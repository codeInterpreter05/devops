# Day 29 — Terraform Fundamentals: HCL Syntax & Providers

**Phase:** 1 – Core DevOps | **Week:** W5 | **Domain:** Terraform | **Flag:** ⚡ Interview-critical

## Brief

Terraform is the tool that turned infrastructure into something you write, review, and version like application code. Before Terraform (and tools like it), "provisioning a VPC" meant a runbook of console clicks that nobody could reliably reproduce. HCL (HashiCorp Configuration Language) is the declarative language you use to describe *desired end state* — you say "I want a VPC with these subnets," not "click here, then here." This file covers the language itself: resources, variables, outputs, locals, and how Terraform talks to cloud APIs through providers.

This day is split into three files:

1. **This file** — HCL syntax fundamentals and providers.
2. **[02-README-State-File.md](02-README-State-File.md)** — the state file: what it is, why it's the single most misunderstood/mismanaged part of Terraform, and workspaces.
3. **[03-README-Terraform-Workflow.md](03-README-Terraform-Workflow.md)** — the `plan`/`apply`/`destroy` workflow and the tfenv/OpenTofu tooling landscape.

## Declarative vs. imperative — the core mental shift

A bash script that provisions infrastructure is **imperative**: "create this, then create that, then attach it." If it fails halfway, you're in an unknown state and re-running it might error on things that already exist. Terraform is **declarative**: you describe the end state in `.tf` files, and Terraform's engine diffs that against what it currently believes exists (the state file) and computes a plan to reconcile the difference. Run `apply` twice with no changes and nothing happens — that **idempotency** is the entire value proposition.

## Anatomy of an HCL file

```hcl
# providers.tf
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

# variables.tf
variable "aws_region" {
  type        = string
  description = "AWS region to deploy into"
  default     = "ap-south-1"
}

variable "vpc_cidr" {
  type        = string
  description = "CIDR block for the VPC"
}

# locals.tf
locals {
  name_prefix = "day29-${terraform.workspace}"
  common_tags = {
    Project   = "devops-roadmap"
    ManagedBy = "terraform"
  }
}

# main.tf
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  tags = merge(local.common_tags, { Name = "${local.name_prefix}-vpc" })
}

# outputs.tf
output "vpc_id" {
  value       = aws_vpc.main.id
  description = "ID of the created VPC"
}
```

Terraform doesn't care about file names or how many files you split things across — it loads **every `.tf` file in the working directory** and merges them into one configuration. The `providers.tf` / `variables.tf` / `main.tf` / `outputs.tf` split is a convention, not a requirement, but it's the convention every team expects.

### Resources — the core building block

```hcl
resource "<PROVIDER_TYPE>" "<LOCAL_NAME>" {
  argument = value
}
```

- `<PROVIDER_TYPE>` (`aws_vpc`, `aws_instance`) tells Terraform which provider and which API object.
- `<LOCAL_NAME>` (`main`) is a **local reference name inside this configuration only** — it is not the AWS resource's `Name` tag and has no meaning outside Terraform.
- Reference a resource's attributes elsewhere with `<TYPE>.<LOCAL_NAME>.<ATTRIBUTE>`, e.g. `aws_vpc.main.id`. This is how Terraform builds its **dependency graph** — referencing `aws_vpc.main.id` inside a subnet resource tells Terraform "create the VPC first."

### Variables — inputs, with types that catch mistakes early

```hcl
variable "subnet_count" {
  type    = number
  default = 2
}

variable "azs" {
  type = list(string)
}

variable "instance_config" {
  type = object({
    instance_type = string
    ami           = string
  })
}
```

Typed variables aren't decoration — `terraform plan` will refuse to run if you pass a string where a `number` is declared, catching typos before they hit a cloud API and cost you a failed `apply` halfway through creating resources.

Precedence for setting a variable's value (highest wins): `-var` / `-var-file` on the CLI > `*.auto.tfvars` > `terraform.tfvars` > environment variables (`TF_VAR_name`) > the `default` in the variable block.

### Outputs — the config's return values

```hcl
output "subnet_ids" {
  value       = aws_subnet.public[*].id
  description = "IDs of all public subnets"
  sensitive   = false
}
```

Outputs are how a root module exposes values to whoever runs it (a human via `terraform output`, or another Terraform config via `terraform_remote_state` — see Day 30). Mark anything secret-like with `sensitive = true` so Terraform redacts it from plan/apply console output (it's still stored in cleartext in the state file — that's a Day 30 topic).

### Locals — named expressions, not inputs

```hcl
locals {
  az_count = length(var.azs)
  is_prod  = var.environment == "prod"
}
```

The difference from variables: **locals aren't set by the caller** — they're computed once, inside the config, to avoid repeating an expression everywhere (DRY within a single module). Variables are the module's *interface*; locals are its *scratch space*.

## Providers & version locking

A **provider** is a plugin that translates HCL into calls against a specific API — `aws`, `google`, `azurerm`, `kubernetes`, even `github` or `datadog`. Terraform itself knows nothing about AWS; the `hashicorp/aws` provider does.

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"   # allows 5.x, blocks 6.0
    }
  }
}
```

Why version-lock providers:
- Provider releases can introduce **breaking changes** in how resources are represented (attribute renames, new required fields) — an unpinned `apply` in CI six months from now might behave completely differently from what you tested.
- `terraform init` writes the *exact* resolved versions into `.terraform.lock.hcl` — **commit this file**. It's the provider equivalent of a `package-lock.json`/`Gemfile.lock`: without it, two engineers (or your laptop vs. CI) can silently resolve different provider versions and get different behavior from *identical* `.tf` source.

## Points to Remember

- Terraform loads every `.tf` file in a directory as one merged configuration — file names are a convention, not a mechanism.
- A resource's local name (`aws_vpc.main`) is only meaningful inside Terraform; it has nothing to do with the cloud object's actual name/tags.
- Referencing one resource's attribute from another (`aws_vpc.main.id`) is what builds Terraform's dependency graph — it's *implicit* dependency, more idiomatic than the explicit `depends_on`.
- Variables are the module's external interface (typed, settable by the caller); locals are internal, computed, DRY helpers.
- Always commit `.terraform.lock.hcl` — it pins the exact provider versions resolved, just like a language package manager's lockfile.

## Common Mistakes

- Not pinning provider versions (`version = ">= 5.0"` with no upper bound, or omitting the block entirely) and getting a different, possibly breaking, provider version months later when a new engineer runs `terraform init` on a machine with no cache.
- Confusing a resource's Terraform local name with its cloud-side `Name` tag — searching the AWS console for "main" and being confused when it's not there.
- Using `depends_on` everywhere out of habit/caution instead of expressing dependencies through attribute references — makes the graph harder to reason about and usually signals the config's actual data flow isn't being modeled.
- Treating `sensitive = true` on an output as "this secret is safe now" — it only hides the value from CLI output; it's still stored in plaintext in the state file (see Day 30 for why that matters for remote state access control).
- Forgetting `.terraform.lock.hcl` is meant to be committed and adding it to `.gitignore` alongside the `.terraform/` provider binary cache (which *should* be gitignored).
