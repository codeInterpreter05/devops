# Day 30 — Terraform Remote State & Modules: Modules & Variable Validation

**Phase:** 1 – Core DevOps | **Week:** W5 | **Domain:** Terraform | **Flag:** ⚡ Interview-critical

## Brief

Copy-pasting the same VPC/EC2 block into five different `.tf` files for five environments is how Terraform configs rot — one gets updated, four don't, and now nobody trusts any of them. **Modules** are Terraform's unit of reuse: a self-contained, parameterized bundle of resources you call like a function. Combined with typed and *validated* variables, modules let you build a small library of "the way we do networking here" that every environment calls consistently. This is the difference between "I wrote some Terraform" and "I designed a Terraform codebase" — a distinction interviewers probe for directly.

## What a module actually is

Every Terraform configuration is already a module — the directory you run `terraform apply` in is the **root module**. A **child module** is just another directory of `.tf` files that you call from elsewhere with a `module` block:

```hcl
# modules/vpc/main.tf
variable "cidr_block" { type = string }
variable "name"       { type = string }

resource "aws_vpc" "this" {
  cidr_block = var.cidr_block
  tags       = { Name = var.name }
}

output "vpc_id" { value = aws_vpc.this.id }
```

```hcl
# root main.tf
module "vpc" {
  source     = "./modules/vpc"
  cidr_block = "10.0.0.0/16"
  name       = "prod-vpc"
}

resource "aws_subnet" "public" {
  vpc_id     = module.vpc.vpc_id     # reference a module's output
  cidr_block = "10.0.1.0/24"
}
```

The module's `variable` blocks are its **inputs**; its `output` blocks are its **return values**, accessed as `module.<NAME>.<OUTPUT>`. Everything declared inside the module (resource local names, internal locals) is scoped to that module — no collision with the root or other modules, which is exactly what lets you call the same module twice (`module "vpc_dev"` and `module "vpc_prod"`) without name clashes.

### Module sources

```hcl
module "vpc" {
  source  = "./modules/vpc"                              # local path
}
module "vpc" {
  source  = "git::https://github.com/org/tf-modules.git//vpc?ref=v1.2.0"  # git, pinned to a tag
}
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"                # Terraform Registry
  version = "~> 5.0"
}
```

Pin git/registry module sources to a specific tag/version the same way you pin providers — an un-pinned `source` pointing at a branch means "this module's behavior can change under me between runs," which is exactly the kind of surprise a production apply should never have.

### Why modules, concretely

- **DRY across environments**: dev/staging/prod call the same `vpc` module with different variable values instead of maintaining three copies of near-identical HCL.
- **Encapsulation**: consumers of the module don't need to know it internally creates 6 resources (VPC, route table, IGW, NAT gateway, subnets, associations) — they see a clean interface (`cidr_block` in, `vpc_id`/`subnet_ids` out).
- **Testability/review surface**: a well-scoped module is small enough to review and reason about completely, unlike a 2,000-line root `main.tf`.
- **Terraform Registry ecosystem**: community modules like `terraform-aws-modules/vpc/aws` encode years of collective "gotchas already handled" — often worth using instead of reinventing, especially for common patterns like VPCs.

## Input validation with variable types

Basic types (`string`, `number`, `bool`, `list(T)`, `map(T)`, `object({...})`) catch *shape* mistakes at plan time. **Validation blocks** catch *value* mistakes — things a type alone can't express:

```hcl
variable "environment" {
  type = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment must be one of: dev, staging, prod."
  }
}

variable "instance_type" {
  type = string
  validation {
    condition     = can(regex("^t3\\.|^m5\\.", var.instance_type))
    error_message = "instance_type must be a t3.* or m5.* family instance."
  }
}

variable "vpc_cidr" {
  type = string
  validation {
    condition     = can(cidrhost(var.vpc_cidr, 0))
    error_message = "vpc_cidr must be a valid CIDR block."
  }
}
```

This moves error detection from "someone discovers it mid-`apply` against a real cloud API, possibly after partially creating resources" to "`terraform plan` refuses immediately, with a message written by the module author explaining exactly what's wrong." For a module used by other teams, this is the difference between a good and a frustrating developer experience — validation errors are the module's contract, enforced automatically.

## Points to Remember

- The directory you run Terraform in is the root module; anything called via a `module` block is a child module with its own scoped resource names.
- A module's `variable` blocks are its inputs; its `output` blocks are its return values, accessed via `module.<name>.<output>`.
- Pin module `source` versions (git tags, registry `version` constraints) exactly like provider versions — an unpinned module source is a silent, moving target.
- Type constraints catch *shape* errors (wrong type entirely); `validation` blocks catch *value* errors (right type, nonsensical value) with a custom, author-written error message.
- The Terraform Registry has mature, widely-used community modules (e.g., `terraform-aws-modules/vpc/aws`) — worth evaluating before writing common patterns like VPCs from scratch.

## Common Mistakes

- Writing a module with no `variable` validation at all, so a typo'd environment name or malformed CIDR isn't caught until AWS's own API rejects it mid-apply — after some resources already got created.
- Not pinning a git-sourced module's `ref`, so `terraform init -upgrade` (or a fresh clone) can silently pull in a newer, behaviorally different version of the module.
- Over-modularizing early — wrapping every single resource in its own module "for reusability" before there's a second consumer, adding indirection with no real DRY benefit yet.
- Under-modularizing — one giant root module shared by every environment with a wall of `count`/`for_each` conditionals keyed off `terraform.workspace`, instead of a clean module boundary with explicit variables per environment.
- Forgetting that changing a module's internal resource names (even without changing its interface) can force `destroy`/`recreate` for consumers on the next `apply`, since Terraform's addressing is derived from resource names inside the module.
