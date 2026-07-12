# Day 56 — Cheatsheet: Terraform Associate Exam Prep

## Core workflow

```bash
terraform init                 # download providers/modules, configure backend
terraform plan                 # compute diff between HCL and refreshed real state
terraform apply                # execute the diff, write new state
terraform destroy              # tear down everything Terraform tracks
terraform plan -out=tfplan     # save an exact plan to apply later
terraform apply tfplan         # apply that exact saved plan (no surprise drift)
```

## State subcommands

```bash
terraform state list                     # every tracked resource address
terraform state list 'aws_instance.*'    # filtered
terraform state show <addr>              # full attributes of one resource

terraform state mv <old_addr> <new_addr>     # rename/move without destroy+recreate
terraform state rm <addr>                    # stop tracking; real object untouched

terraform state pull > backup.tfstate        # download state as JSON
terraform state push backup.tfstate          # overwrite remote state (dangerous)

terraform state replace-provider <from> <to>  # rewrite provider addresses in state
```

## `import`

```bash
terraform import <resource_addr> <real_id>
# then IMMEDIATELY:
terraform plan     # non-empty diff = your HCL doesn't match reality yet, fix HCL not infra
```

## `taint` / `-replace`

```bash
terraform taint <addr>          # legacy: mutates state immediately, no preview
terraform apply

terraform plan -replace="<addr>"    # modern: shows up in plan, reviewable
terraform apply -replace="<addr>"
```

## `graph`, `fmt`, `validate`

```bash
terraform graph | dot -Tpng > graph.png   # needs graphviz's `dot`

terraform fmt                 # rewrite to canonical style
terraform fmt -recursive      # include subdirectories
terraform fmt -check          # CI-safe: exit non-zero if changes needed, no writes
terraform fmt -diff           # show diff without writing

terraform validate            # local syntax/type/reference check — NO provider calls, no creds needed
```

## Remote backend (S3 example)

```hcl
terraform {
  backend "s3" {
    bucket         = "my-tf-state-bucket"
    key            = "prod/network/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "tf-state-lock"   # locking (pre-1.10 pattern; S3-native locking exists 1.10+)
    encrypt        = true
  }
}
```

```hcl
data "terraform_remote_state" "network" {
  backend = "s3"
  config  = { bucket = "my-tf-state-bucket", key = "prod/network/terraform.tfstate", region = "us-east-1" }
}
# usage: data.terraform_remote_state.network.outputs.<name>
```

## Terraform Cloud / Enterprise vs OSS

| Feature | OSS | TFC | TFE |
|---|---|---|---|
| Remote state + locking | manual (self-managed backend) | built-in | built-in |
| Remote execution | no | yes (hosted) | yes (self-hosted) |
| Policy as code (Sentinel/OPA) | no | paid tiers | yes |
| VCS-driven runs | no | yes | yes |
| RBAC / SSO | no | paid tiers | yes |

## Sentinel enforcement levels

```
advisory        -> warn only, never blocks
soft-mandatory  -> blocks, but an authorized user can override
hard-mandatory  -> blocks, no override
```

## The 9 exam objective domains (checklist)

```
1. IaC concepts
2. Terraform's purpose vs other IaC
3. Terraform basics (providers, resources, data sources, vars, outputs)
4. CLI outside core workflow (fmt, validate, taint/-replace, import, graph, workspace)
5. Modules (sourcing, versioning, registry)
6. Core workflow (init/plan/apply/destroy)
7. State (backends, locking, sensitive data, remote_state)
8. Configuration (HCL, meta-args: count/for_each/depends_on/lifecycle, functions)
9. Terraform Cloud & Enterprise capabilities
```
