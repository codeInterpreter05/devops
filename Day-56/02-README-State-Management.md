# Day 56 — Terraform Associate Exam Prep: State Management Deep Dive

**Phase:** 1 – Core DevOps | **Week:** W9 | **Domain:** Terraform | **Flag:** 📌

## Brief

State is the single most exam-tested, interview-tested, and production-incident-tested topic in all of Terraform. `terraform.tfstate` is the file that maps your HCL's resource blocks to real-world object IDs — without it, Terraform has no way to know that `aws_instance.web` *is* `i-0abc123...` in your AWS account, and every `plan` would think everything needs to be created from scratch. Nearly every "Terraform Associate practice exam, focus on state management" prompt exists because state is where theoretical Terraform knowledge meets real operational pain: locking, drift, corrupted state files, and moving resources between configurations.

## What state actually is, and why it must exist

Terraform's core design is: **HCL describes desired state; the provider APIs are the source of truth for actual state; the `.tfstate` file is Terraform's cached mapping between the two**, plus metadata (dependency graph info, resource attributes not knowable from HCL alone, like an auto-generated ID or ARN). On every `plan`/`apply`, Terraform:

1. Refreshes state by querying real provider APIs for each tracked resource (unless `-refresh=false`).
2. Compares refreshed real-world state against desired HCL state.
3. Computes a diff (the "plan").
4. On `apply`, executes the diff and writes the new state back.

This is why deleting `terraform.tfstate` doesn't delete your infrastructure — but it does make Terraform "forget" it ever created anything, so the next `plan` will propose creating everything again (and likely fail with "already exists" errors from the cloud provider, since the real resources are still there).

## `terraform state` subcommands

```bash
terraform state list                          # list every resource address currently tracked
terraform state list 'aws_instance.*'         # filter by pattern

terraform state show aws_instance.web         # dump all attributes Terraform knows for one resource

terraform state mv aws_instance.web aws_instance.web_server
# renames a resource address WITHOUT destroying/recreating the real object —
# use when you refactor HCL (rename a resource, or move it into/out of a module)

terraform state rm aws_instance.web
# removes a resource from state WITHOUT destroying the real infrastructure —
# Terraform simply stops tracking it; the object keeps running, unmanaged

terraform state pull > backup.tfstate         # download current state as JSON, e.g. for manual inspection/backup
terraform state push backup.tfstate           # upload a local state file to the configured backend (dangerous — overwrites remote state)

terraform state replace-provider registry.terraform.io/-/aws registry.terraform.io/hashicorp/aws
# rewrites provider addresses in state, e.g. after a provider namespace change
```

**`state mv` is the one that separates people who've refactored real Terraform codebases from people who haven't.** If you rename a resource in HCL from `aws_instance.web` to `aws_instance.web_server` and just run `terraform apply`, Terraform sees the old address is gone and the new one is new — it will **destroy** the old instance and **create** a new one, because from Terraform's perspective these are two unrelated resource addresses. `terraform state mv` tells Terraform "this is the same real object, just a new address," avoiding the destroy/recreate entirely. The same applies when moving a resource into a module (`aws_instance.web` → `module.compute.aws_instance.web`).

## Remote backends and locking

Local state (a `terraform.tfstate` file on your laptop) is fine for a solo learning project and dangerous for anything real: no locking (two people running `apply` simultaneously can corrupt state or double-apply), no shared visibility, and a single lost laptop = lost state.

```hcl
terraform {
  backend "s3" {
    bucket         = "my-tf-state-bucket"
    key            = "prod/network/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "tf-state-lock"   # provides locking (pre-Terraform 1.10 pattern)
    encrypt        = true
  }
}
```

The DynamoDB table implements **state locking**: before writing state, Terraform writes a lock record to DynamoDB; if another `apply` is already running, the second one fails fast with a clear "state is locked" error instead of racing and corrupting the state file. (As of Terraform 1.10+, S3 natively supports locking via conditional writes without a separate DynamoDB table — know both patterns, since most existing production code still uses the DynamoDB approach.)

`terraform_remote_state` lets one configuration read another configuration's outputs — the standard pattern for splitting a large infrastructure into independently-applied layers (e.g., a `network` state that a `compute` state reads from):

```hcl
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "my-tf-state-bucket"
    key    = "prod/network/terraform.tfstate"
    region = "us-east-1"
  }
}

resource "aws_instance" "web" {
  subnet_id = data.terraform_remote_state.network.outputs.public_subnet_id
}
```

## Sensitive data in state — the uncomfortable truth

**State files store resource attributes in plain text, including anything marked `sensitive = true` in your HCL.** The `sensitive` flag only affects what Terraform *prints* in CLI output — it does not encrypt or redact the value inside `terraform.tfstate` itself. A `aws_db_instance` resource's `password` attribute, for example, sits in the state file unencrypted unless the backend itself encrypts data at rest (S3 with `encrypt = true` + SSE, or TFC/TFE which encrypt state server-side).

This is why: state files must never be committed to git, must be stored in a backend with encryption at rest, and access to the state backend should be treated as equivalent to access to production secrets.

## Points to Remember

- State is Terraform's cached mapping of HCL resource addresses to real-world object IDs — the provider API is truth, state is the cache, refreshed on every plan.
- `terraform state mv` preserves a real object across a resource-address rename/module-move — skipping it causes an unwanted destroy+recreate.
- `terraform state rm` stops tracking a resource without touching the real infrastructure — used when you want to "hand off" a resource to be managed elsewhere or manually.
- Remote backends (S3, TFC, etc.) exist primarily for shared access + locking — local state has neither and is a single point of failure/corruption for team use.
- `sensitive = true` hides values from CLI output only — it does **not** encrypt the state file; encryption at rest is the backend's job.
- `terraform_remote_state` is the standard pattern for one Terraform configuration/layer to consume another's outputs.

## Common Mistakes

- Renaming a resource in HCL and running `apply` directly, causing an unintended destroy+recreate of a real, possibly stateful resource (a database, a load balancer with a fixed DNS record) instead of using `terraform state mv` first.
- Assuming `sensitive = true` means the value is encrypted or absent from state — it's neither; anyone with read access to the state file/backend can see it in plain text.
- Committing `terraform.tfstate` to version control (common in tutorials/first projects) — instantly leaks every resource attribute, including secrets, to anyone with repo access and git history.
- Two engineers running `terraform apply` at the same time against local or unlocked state, corrupting the file or causing conflicting real-world changes.
- Manually editing `.tfstate` JSON by hand instead of using `terraform state` subcommands — easy to introduce a subtle inconsistency that only surfaces on the next `plan`/`apply` as a confusing error.
- Running `terraform state push` casually — it fully overwrites remote state and can silently wipe out changes another teammate applied since you pulled.
