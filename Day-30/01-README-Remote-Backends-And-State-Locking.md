# Day 30 — Terraform Remote State & Modules: Remote Backends & State Locking

**Phase:** 1 – Core DevOps | **Week:** W5 | **Domain:** Terraform | **Flag:** ⚡ Interview-critical

## Brief

Yesterday you learned why a local `terraform.state` file is fragile: no locking, no sharing, plaintext secrets sitting in a file that could vanish with your laptop. A **remote backend** fixes exactly this — it moves state to shared, durable storage and adds locking so two engineers (or two CI runs) can never corrupt it by applying at the same time. The canonical AWS pattern — S3 for storage + DynamoDB for locking — is one of the most commonly asked "have you actually run Terraform on a team" interview questions, because getting it wrong (or not knowing it exists) is a fast way to lose infrastructure trust.

This day is split into three files:

1. **This file** — remote backends, S3 + DynamoDB locking, and `terraform_remote_state`.
2. **[02-README-Modules-And-Variable-Validation.md](02-README-Modules-And-Variable-Validation.md)** — module structure/reuse and input validation.
3. **[03-README-Outputs-And-Terraform-Cloud.md](03-README-Outputs-And-Terraform-Cloud.md)** — output dependency chains and Terraform Cloud as a managed alternative.

## Why a local state file doesn't scale past one person

With local state, "sharing infrastructure with a team" means either (a) only one person ever runs `apply` (a bottleneck and single point of failure), or (b) everyone copies the state file around manually (guaranteed eventual corruption/staleness — whoever applies last silently overwrites everyone else's view). Remote backends solve this by making state a **shared, centrally-stored, lockable resource** instead of a file on someone's laptop.

## The S3 + DynamoDB pattern

```hcl
terraform {
  backend "s3" {
    bucket         = "my-org-terraform-state"
    key            = "day30/terraform.tfstate"
    region         = "ap-south-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

- **S3 bucket** stores the actual state file (`terraform.tfstate`) as an object at the given `key`. Enable **versioning** on the bucket — this gives you the "restore a previous state" safety net that local state's `.tfstate.backup` only weakly approximates. Enable **SSE (server-side encryption)** since state contains plaintext secrets — `encrypt = true` in the backend block handles this for the write path.
- **DynamoDB table** provides **locking**. Terraform writes a lock record (item) into the table before starting `plan`/`apply`, keyed by the state's S3 path. If a second `apply` starts while the lock exists, it fails fast with a clear "state locked" error instead of racing the first one and corrupting state. The table needs exactly one attribute: a primary key named `LockID` (string type) — Terraform manages the rest.

```bash
aws dynamodb create-table \
  --table-name terraform-locks \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```

**Note (as of Terraform 1.6+/1.10):** newer Terraform versions support **S3 native locking** using S3's own conditional-write (`If-None-Match`) feature, removing the DynamoDB dependency entirely for new setups. The DynamoDB pattern remains extremely common because most production setups predate this and haven't migrated — know both, but understand *why* locking is needed regardless of mechanism.

### What happens without locking

Two engineers both run `terraform apply` at nearly the same moment against the same state. Both read the same "current state," both compute a plan based on it, and both write their own updated state back — whoever finishes last wins, silently discarding the other's changes from state's perspective (even though both sets of changes may have actually happened against real infrastructure). This is exactly the corruption scenario DynamoDB locking exists to prevent — it forces the second `apply` to wait or fail instead of racing.

## `terraform_remote_state` — reading another config's outputs

```hcl
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "my-org-terraform-state"
    key    = "day29/terraform.tfstate"
    region = "ap-south-1"
  }
}

resource "aws_instance" "app" {
  subnet_id = data.terraform_remote_state.network.outputs.subnet_ids[0]
}
```

This is how separate Terraform configs (often owned by separate teams — networking vs. application) compose: the networking config's `outputs.tf` becomes the application config's input, read directly out of the networking config's *state* rather than hardcoding IDs. The tradeoff: whoever reads via `terraform_remote_state` needs **read access to the source config's entire state file** — including any of its non-sensitive-looking-but-actually-sensitive data. This is one reason many teams prefer a dedicated **module + explicit outputs + a proper data-sharing convention (SSM Parameter Store, Terraform Cloud's `run` triggers, or plain shared variables)** over widescale `terraform_remote_state` reads once an org grows past a couple of teams.

## Points to Remember

- Remote backends (e.g., S3) give you shared, durable, versioned state storage; DynamoDB (or newer S3-native locking) prevents concurrent applies from corrupting it.
- Enable S3 bucket versioning and encryption on any state bucket — versioning is your real state backup mechanism; encryption matters because state holds plaintext secrets.
- A DynamoDB lock table needs just one key: a string primary key named `LockID` — Terraform manages everything else.
- `terraform_remote_state` reads another config's full state to access its outputs — treat granting access to it as granting access to everything in that state file, not just the specific output you need.
- Migrating from local to remote backend (or between backends) is done via `terraform init -migrate-state` — always back up the local state file first.

## Common Mistakes

- Setting up an S3 backend without enabling bucket versioning, then losing the ability to roll back after a bad `apply` corrupts state.
- Forgetting to also configure a lock table (or S3 native locking) — teams sometimes stop at "state is in S3 now" and assume that alone prevents concurrent-apply corruption; it doesn't, S3 storage and locking are separate concerns.
- Granting broad `terraform_remote_state` read access across teams as a shortcut, inadvertently exposing another team's sensitive outputs (DB passwords, internal IPs) to anyone who can read that state.
- Hardcoding the backend `bucket`/`key`/`region` identically across environments, causing dev and prod to silently share (and overwrite) the same state file — the `key` must be unique per environment/config.
- Forgetting that changing backend configuration requires `terraform init -migrate-state` — just editing the `backend` block and running `plan` leaves Terraform confused about where state actually lives.
