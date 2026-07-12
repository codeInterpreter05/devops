# Day 29 — Resources: Terraform Fundamentals

## Primary (assigned)

- **HashiCorp Learn: Terraform — Getting Started (AWS)** (developer.hashicorp.com/terraform/tutorials/aws-get-started) — free, official, the assigned starting point. Walks through `init`/`plan`/`apply`/`destroy` against real AWS resources exactly as this lab does.

## Deepen your understanding

- **Terraform Language documentation** (developer.hashicorp.com/terraform/language) — the authoritative reference for HCL syntax, variable types, and expressions; go here whenever the tutorial glosses over a detail.
- **"Terraform: Up & Running" by Yevgeniy Brikman** — the canonical book on Terraform in production; Chapters 1-2 map directly to today's material (later chapters are Day 30's assigned reading).
- **HashiCorp — "Purpose of Terraform State"** (developer.hashicorp.com/terraform/language/state/purpose) — short, focused doc explaining exactly why state exists and what it's used for.

## Reference / lookup

- **Terraform AWS Provider docs** (registry.terraform.io/providers/hashicorp/aws/latest/docs) — the single most-used reference for looking up any `aws_*` resource's arguments and attributes.
- **`terraform -help`** and **`terraform <subcommand> -help`** — every command's flags, on-box, no internet needed.

## Practice

- **HashiCorp Learn — "Terraform Workspaces" tutorial** — hands-on practice with the same workspace behavior covered in Lab 4, reinforcing that workspaces isolate state, not config.
