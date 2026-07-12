# Day 30 — Resources: Terraform Remote State & Modules

## Primary (assigned)

- **"Terraform: Up & Running" by Yevgeniy Brikman — Chapters 1-3** — the assigned reading. Chapter 3 in particular ("How to Manage Terraform State") is the best single treatment of why remote state + locking matters, written by someone who's operated it at scale.

## Deepen your understanding

- **HashiCorp — "Backend Type: s3"** (developer.hashicorp.com/terraform/language/settings/backends/s3) — the authoritative reference for every S3 backend argument, including the newer native S3-locking option.
- **HashiCorp — "Modules" documentation** (developer.hashicorp.com/terraform/language/modules) — covers module sources, versioning, and the difference between root and child modules in full depth.
- **Terraform Registry — `terraform-aws-modules/vpc/aws`** (registry.terraform.io) — read the source of a mature, widely-used community VPC module to see real-world module design (variable validation, sensible defaults, extensive outputs).
- **HashiCorp — "Refactoring" / `moved` blocks documentation** — covers `terraform state mv` and the newer declarative `moved` block for refactors that shouldn't destroy/recreate resources, directly relevant to Lab 3.

## Reference / lookup

- **Terraform Cloud documentation** (developer.hashicorp.com/terraform/cloud-docs) — for when you need the managed-backend feature list precisely.
- `terraform state -help` and `terraform init -help` — on-box reference for every state/backend subcommand.

## Practice

- **HashiCorp Learn — "Terraform Cloud Get Started" tutorial** — free tier walkthrough, useful for comparing the managed experience directly against the DIY S3+DynamoDB setup from today's lab.
