# Day 29 — Terraform Fundamentals: The State File

**Phase:** 1 – Core DevOps | **Week:** W5 | **Domain:** Terraform | **Flag:** ⚡ Interview-critical

## Brief

If HCL is the language, **state** is Terraform's memory. This is the single most asked-about, most misunderstood, and most dangerous-to-mishandle part of Terraform — and it's exactly the kind of thing that separates someone who's typed `terraform apply` a few times from someone who's actually operated Terraform on a team. Get comfortable with what state is, why it exists, and workspaces (a lightweight way to reuse the same config for multiple environments) before you touch remote backends in Day 30.

## What the state file actually is

`terraform.state` is a JSON file that maps every resource in your `.tf` config to the **real-world object it corresponds to** (an AWS VPC ID, an EC2 instance ID, etc.) plus a cached copy of that object's last-known attributes.

```bash
terraform show -json | jq '.values.root_module.resources[0]'
```

Why Terraform needs this at all: cloud APIs don't give you a way to ask "which of my 400 EC2 instances did *this specific Terraform config* create?" Terraform can't infer ownership from tags alone (tags can collide, be missing, or be edited by hand). State is Terraform's own bookkeeping — the source of truth for **"what do I manage, and what did I last see its attributes as."**

### What state is used for, concretely

1. **Mapping** — resource address (`aws_instance.web`) → real ID (`i-0abc123...`).
2. **Metadata cache** — avoids an API call for every attribute reference; `aws_vpc.main.cidr_block` used elsewhere in the config is read from state, not re-fetched, until the next `refresh`/`plan`.
3. **Dependency ordering** — state records the dependency graph so `destroy` can tear things down in reverse-creation order.
4. **Diffing** — `terraform plan` computes "add/change/destroy" by comparing: (a) your `.tf` config (desired), (b) the state file (last-known), and (c) a live refresh against the actual API (current reality) — you're diffing potentially three different things, not two.

### Why the state file is dangerous

- **It contains secrets in plaintext.** Any attribute that touches a resource — a generated RDS password, a TLS private key, a `sensitive` output's actual value — sits unencrypted in the JSON unless the backend itself encrypts at rest (e.g., S3 with SSE). Never commit `terraform.state` to git.
- **It's a single point of truth with no built-in merge.** Two people running `apply` against the same *local* state file concurrently will corrupt it — Terraform doesn't merge state changes like git merges text.
- **Losing it is catastrophic.** If `terraform.state` is deleted and you have no backup, Terraform no longer knows what it manages. Your infrastructure still exists in AWS, but Terraform "forgets" it and would try to create duplicates on the next `apply`. Recovery means painstakingly `terraform import`-ing every resource back in, one at a time.
- **Drift**: if someone changes a resource by hand in the console (or another tool touches it), state goes stale until the next `plan`/`refresh` notices the difference. This is one of the most common real-world sources of "terraform wants to destroy and recreate something for no reason I changed."

### `terraform.tfstate.backup`

Terraform automatically keeps one previous version of local state as `terraform.tfstate.backup` before overwriting it — a shallow safety net for local-state workflows, not a substitute for real backups/versioning (which is exactly what remote backends with versioning, covered Day 30, solve properly).

## Workspaces — one config, multiple state files

```bash
terraform workspace new staging
terraform workspace new prod
terraform workspace list
terraform workspace select staging
terraform apply   # uses staging's own state file
```

A **workspace** is Terraform's built-in mechanism for using the *same* `.tf` configuration to manage *multiple, independent* sets of infrastructure — most commonly one workspace per environment (dev/staging/prod). Internally, each workspace just gets its own state file (locally: `terraform.tfstate.d/<workspace>/terraform.tfstate`; in an S3 backend: a different key path per workspace).

```hcl
resource "aws_instance" "web" {
  instance_type = terraform.workspace == "prod" ? "m5.large" : "t3.micro"
  tags = { Environment = terraform.workspace }
}
```

**The interview-critical caveat:** workspaces only change *which state file* you're pointed at — they do **not** change which `.tf` files, provider config, or variable *values* are loaded automatically. If prod and dev need genuinely different variable values (different accounts, different CIDR ranges), you still need `-var-file=prod.tfvars` per workspace, or you'll `apply` prod's workspace with dev's variable defaults. For meaningfully different environments (different AWS accounts, different compliance requirements), most teams outgrow workspaces and switch to **separate root modules per environment** (or Terragrunt — Day 31) precisely because workspaces don't isolate *configuration*, only *state*.

## Points to Remember

- State maps Terraform's config to real cloud resource IDs — without it, Terraform has no idea what it owns.
- State contains **plaintext secrets** — never commit it to version control, and treat access to the state backend as access to production credentials.
- `terraform plan` diffs three things: your `.tf` config, the last-known state, and a live refresh of actual cloud state — drift shows up when the last one disagrees with the second.
- Losing the state file doesn't destroy your infrastructure, but it does make Terraform "forget" it manages that infrastructure — recovery requires `terraform import` per resource.
- Workspaces isolate **state**, not **configuration** — they're a lightweight tool for near-identical environments, not a full multi-environment strategy on their own.

## Common Mistakes

- Committing `terraform.tfstate` (or `.tfstate.backup`) to a git repo — instantly leaks every secret attribute Terraform has ever touched, and git history means "delete the file" doesn't actually remove the exposure.
- Manually editing resources in the AWS console that Terraform manages, then being surprised when the next `plan` wants to "fix" (revert) the manual change — Terraform's state is the truth it reconciles toward, not the live console.
- Assuming `terraform destroy` is reversible because "the state file still has the old values" — once a resource is destroyed in the cloud and removed from state, the metadata is gone; state is not a backup/rollback mechanism.
- Relying on workspaces alone to separate dev and prod, then discovering both share the same hardcoded values in `main.tf` because nobody realized workspaces don't swap variable files automatically.
- Running `terraform apply` from two different laptops against the same local state file at the same time — no locking exists on local state, so this can corrupt it (this is precisely the problem remote backends with locking solve — Day 30).
