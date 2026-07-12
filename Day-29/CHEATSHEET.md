# Day 29 — Cheatsheet: Terraform Fundamentals

## Core workflow

```bash
terraform init                 # download providers/modules, set up backend
terraform init -upgrade         # allow providers to move within version constraints
terraform fmt                    # rewrite files to canonical formatting
terraform fmt -check              # CI mode: exit non-zero if formatting is wrong, don't rewrite
terraform validate                 # syntax + internal consistency, no API calls
terraform plan                      # compute + show diff, read-only
terraform plan -out=tfplan           # save the plan to a file
terraform apply                       # interactive apply (prompts y/n)
terraform apply tfplan                 # apply an exact saved plan
terraform apply -auto-approve           # no prompt — CI only, never for humans on prod
terraform destroy                        # tear down everything in this config
terraform destroy -target=aws_instance.x  # scoped destroy — use sparingly
```

## HCL building blocks

```hcl
resource "aws_vpc" "main" { ... }        # <TYPE> <LOCAL_NAME>
data "aws_ami" "amazon_linux" { ... }      # read-only lookup, not managed by Terraform

variable "name" {
  type        = string        # string | number | bool | list(T) | map(T) | object({...})
  default     = "value"
  description = "..."
  sensitive   = false
}

output "name" {
  value     = aws_vpc.main.id
  sensitive = false
}

locals {
  computed = "${var.name}-suffix"
}

# reference another resource's attribute (builds the dependency graph)
aws_subnet.public[0].id
aws_vpc.main.cidr_block
```

## Variable value precedence (highest wins)

```
-var / -var-file (CLI flags)
*.auto.tfvars (auto-loaded, alphabetical)
terraform.tfvars (auto-loaded)
TF_VAR_<name> environment variables
default in the variable block
```

## State inspection

```bash
terraform show                     # human-readable current state
terraform show -json | jq .        # machine-readable
terraform state list                 # every resource address Terraform manages
terraform state show aws_vpc.main      # detail on one resource
terraform state mv <old> <new>           # rename in state, no infra change
terraform state rm <addr>                 # stop managing a resource (does NOT destroy it)
terraform import aws_vpc.main vpc-0abc123  # bring an existing resource under management
terraform refresh                          # sync state with real infra (no plan/apply)
```

## Workspaces

```bash
terraform workspace list
terraform workspace new staging
terraform workspace select staging
terraform workspace show
terraform workspace delete staging     # must not be the current workspace
terraform.workspace                     # use inside HCL, e.g. "${terraform.workspace}"
```

## Provider & version pinning

```hcl
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"    # allows 5.x, blocks 6.0
    }
  }
}
```
`.terraform.lock.hcl` — commit this. `.terraform/` — gitignore this.

## Tooling

```bash
tfenv install latest
tfenv install 1.7.5
tfenv use 1.7.5
tfenv list
echo "1.7.5" > .terraform-version   # per-project pin, auto-picked up by tfenv

tofu init / tofu plan / tofu apply   # OpenTofu — same commands, MPL-licensed fork
```
