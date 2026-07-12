# Day 30 — Cheatsheet: Terraform Remote State & Modules

## S3 + DynamoDB backend

```hcl
terraform {
  backend "s3" {
    bucket         = "my-org-terraform-state"
    key            = "envs/prod/terraform.tfstate"
    region         = "ap-south-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

```bash
# bootstrap the backend resources (once)
aws s3api create-bucket --bucket my-org-terraform-state --region ap-south-1 \
  --create-bucket-configuration LocationConstraint=ap-south-1
aws s3api put-bucket-versioning --bucket my-org-terraform-state \
  --versioning-configuration Status=Enabled
aws dynamodb create-table --table-name terraform-locks \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST

terraform init                  # detects new/changed backend, offers migration
terraform init -migrate-state   # explicit migration
terraform init -reconfigure     # ignore old backend config, use new one as-is
```

## State locking behavior

```bash
terraform force-unlock <LOCK_ID>   # manually break a stuck lock (only if you're SURE no one else is applying)
```

## `terraform_remote_state`

```hcl
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "my-org-terraform-state"
    key    = "envs/prod/network/terraform.tfstate"
    region = "ap-south-1"
  }
}
# use it:
subnet_id = data.terraform_remote_state.network.outputs.subnet_ids[0]
```

## Modules

```hcl
module "vpc" {
  source     = "./modules/vpc"          # local
  # source   = "git::https://github.com/org/repo.git//vpc?ref=v1.2.0"
  # source   = "terraform-aws-modules/vpc/aws"
  # version  = "~> 5.0"                 # registry modules only
  cidr_block = "10.0.0.0/16"
}

module.vpc.vpc_id           # access a module's output elsewhere
```

## Variable validation

```hcl
variable "environment" {
  type = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment must be dev, staging, or prod."
  }
}
```

## Refactoring state without destroying infra

```bash
terraform state list                                   # see current addresses
terraform state mv aws_vpc.main module.vpc.aws_vpc.main
terraform state mv 'aws_subnet.public[0]' 'module.vpc.aws_subnet.public[0]'
terraform state rm <addr>       # stop managing, does NOT destroy real resource
terraform import <addr> <id>    # bring an existing resource under management
```

## Dependency ordering

```hcl
depends_on = [aws_iam_role_policy.lambda_policy]   # explicit, use only when no attribute reference exists

output "name" {
  value      = module.app.dns_name
  depends_on = [aws_eip_association.this]           # rare, output-level depends_on
}
```

## Terraform Cloud

```hcl
terraform {
  cloud {
    organization = "my-org"
    workspaces { name = "prod-networking" }
  }
}
```
```bash
terraform login       # authenticate CLI to TFC
```
