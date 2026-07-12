# Day 31 — Cheatsheet: Terraform CI & Advanced Patterns

## Terragrunt

```hcl
# root terragrunt.hcl
remote_state {
  backend = "s3"
  generate = { path = "backend.tf", if_exists = "overwrite" }
  config = {
    bucket = "my-org-tf-state"
    key    = "${path_relative_to_include()}/terraform.tfstate"
    region = "ap-south-1"
    dynamodb_table = "terraform-locks"
    encrypt = true
  }
}
```
```hcl
# child terragrunt.hcl
include "root" { path = find_in_parent_folders() }
terraform { source = "git::https://github.com/org/modules.git//vpc?ref=v1.2.0" }
inputs = { cidr_block = "10.0.0.0/16" }

dependency "vpc" { config_path = "../vpc" }
# used as: dependency.vpc.outputs.vpc_id
```
```bash
terragrunt plan
terragrunt apply
terragrunt run-all plan     # whole tree, dependency-ordered
terragrunt run-all apply
```

## CI: plan on PR, apply on merge (GitHub Actions)

```yaml
on:
  pull_request: { paths: ["infra/**"] }
  push: { branches: [main], paths: ["infra/**"] }

permissions:
  contents: read
  pull-requests: write
  id-token: write        # for OIDC

steps:
  - uses: actions/checkout@v4
  - uses: hashicorp/setup-terraform@v3
  - uses: aws-actions/configure-aws-credentials@v4
    with: { role-to-assume: arn:aws:iam::ACCOUNT:role/gha-role, aws-region: ap-south-1 }
  - run: terraform fmt -check -recursive
  - run: terraform init
  - run: terraform validate
  - run: terraform plan -no-color -out=tfplan       # PR only
  - run: terraform apply -auto-approve tfplan        # main push only
```

## Drift detection

```bash
terraform plan -detailed-exitcode
# exit 0 = no changes   exit 1 = error   exit 2 = drift detected
```

## Checkov

```bash
pip install checkov
checkov -d infra/ --framework terraform
checkov -d infra/ --framework terraform --compact --quiet
checkov -f main.tf                          # single file
```
```hcl
#checkov:skip=CKV_AWS_20:Public website bucket, intentional
resource "aws_s3_bucket" "site" { ... }
```

## OPA / Rego (against plan JSON)

```bash
terraform show -json tfplan > plan.json
conftest test plan.json -p policy/
```
```rego
package main
deny[msg] {
  rc := input.resource_changes[_]
  rc.type == "aws_security_group_rule"
  rc.change.after.cidr_blocks[_] == "0.0.0.0/0"
  rc.change.after.from_port == 22
  msg := sprintf("SSH open to world: %s", [rc.address])
}
```

## Sentinel (Terraform Cloud/Enterprise)

```python
import "tfplan/v2" as tfplan
main = rule {
  all tfplan.resource_changes as _, rc {
    rc.type is not "aws_db_instance" or rc.change.after.publicly_accessible is false
  }
}
```

## Infracost

```bash
infracost breakdown --path infra/
infracost diff --path infra/ --compare-to infracost-base.json
```

## OIDC setup (no static AWS keys in CI)

```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
```

## Atlantis (PR comment-driven, self-hosted)

```
atlantis plan     # comment on PR to trigger
atlantis apply    # comment on PR to apply the plan
```
