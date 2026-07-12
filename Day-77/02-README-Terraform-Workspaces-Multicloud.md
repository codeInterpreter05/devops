# Day 77 — Multi-cloud CI/CD: Terraform Workspaces for Multi-Cloud

**Phase:** 2 – CI/CD & Security | **Week:** W12 | **Domain:** CI/CD

## Brief

Terraform is the piece of tooling that makes "one pipeline, multiple clouds" tractable at the infrastructure layer — but naively reaching for **workspaces** as the mechanism to separate AWS from GCP resources is a common misunderstanding of what workspaces are actually for. This file untangles that: what Terraform workspaces really solve (environment separation within one configuration), how multiple **provider blocks** are what actually let one Terraform codebase target multiple clouds, and how the two concepts combine in a real multi-cloud setup.

## What Terraform workspaces actually are

A Terraform workspace is a **named state file** within the same working directory/configuration — `terraform workspace new staging` creates an isolated state (`terraform.tfstate.d/staging`) without duplicating any `.tf` files. The core use case is running the *same configuration* against multiple instances of the *same infrastructure shape* — classically, `dev`/`staging`/`production` instances of one app's infrastructure, all defined by identical `.tf` files, differing only in variable values (instance size, replica count) and, critically, in *state* (each environment's resources are tracked completely separately).

```bash
terraform workspace new staging
terraform workspace new production
terraform workspace select staging
terraform apply -var-file=staging.tfvars
```

```hcl
resource "aws_instance" "app" {
  instance_type = terraform.workspace == "production" ? "m5.xlarge" : "t3.medium"
  count         = terraform.workspace == "production" ? 3 : 1
}
```

**Workspaces do not switch providers.** They switch *state*, within a configuration that's still targeting whatever provider(s) are declared in that configuration. This is the single most common misconception: "I'll use a workspace per cloud" doesn't work the way people expect, because a workspace has no mechanism to conditionally include/exclude entire provider blocks or resource types based on which workspace is active — you'd need `terraform.workspace == "aws"` conditionals sprinkled through every resource, which quickly becomes unreadable and defeats the purpose.

## What actually lets one Terraform codebase target multiple clouds: multiple provider blocks

Terraform supports declaring more than one provider in the same configuration — this, not workspaces, is the real multi-cloud mechanism:

```hcl
# providers.tf
provider "aws" {
  region = "us-east-1"
}

provider "google" {
  project = "my-gcp-project"
  region  = "us-central1"
}

# main.tf — both providers used in the same apply
resource "aws_lambda_function" "app" {
  function_name = "myapp"
  # ...
}

resource "google_cloud_run_v2_service" "app" {
  name     = "myapp"
  location = "us-central1"
  # ...
}
```

A single `terraform apply` here provisions resources on **both** AWS and GCP simultaneously, tracked in one state file. This is genuinely cloud-agnostic infrastructure-as-code: the same tool, the same workflow (`plan`/`apply`/`destroy`), the same state-locking and drift-detection semantics, regardless of which cloud a given resource lives in.

## Combining workspaces and multi-provider configs correctly

The two concepts aren't mutually exclusive — they solve different axes of variation, and a real multi-cloud, multi-environment setup uses both:

```
Axis 1 (workspace): which environment — dev / staging / production
Axis 2 (provider blocks + resources): which cloud — AWS resources + GCP resources, both declared
```

```bash
terraform workspace select staging
terraform apply   # provisions BOTH AWS Lambda and GCP Cloud Run resources, in the "staging" state
terraform workspace select production
terraform apply   # provisions BOTH again, separately, in the "production" state
```

A cleaner alternative many teams prefer over deeply overloading workspaces: **separate root modules per environment** (`environments/staging/main.tf`, `environments/production/main.tf`), each with its own backend config and its own `module` calls into shared `modules/aws-app` and `modules/gcp-app` modules. This avoids `terraform.workspace == X` conditionals scattered through resource definitions and makes the actual per-environment differences explicit and reviewable in each environment's own small entrypoint file, at the cost of a bit more directory structure.

```
environments/
├── staging/
│   ├── main.tf          # calls module "aws_app" and module "gcp_app" with staging-sized inputs
│   └── backend.tf
└── production/
    ├── main.tf          # calls the same modules with production-sized inputs
    └── backend.tf
modules/
├── aws_app/
└── gcp_app/
```

## Remote state and locking across clouds

A genuinely multi-cloud Terraform setup still needs **one** backend for state storage/locking — you don't need (and shouldn't want) separate state backends per cloud for a single logical environment's configuration. A common pattern: keep Terraform state itself in one cloud (commonly AWS S3 + DynamoDB for locking, since it's mature and cheap) even when the *resources being managed* span multiple clouds — the state backend choice is independent of which providers the configuration declares.

```hcl
terraform {
  backend "s3" {
    bucket         = "myorg-tfstate"
    key            = "multi-cloud-app/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "tf-lock"
    encrypt        = true
  }
  required_providers {
    aws    = { source = "hashicorp/aws" }
    google = { source = "hashicorp/google" }
  }
}
```

## Points to Remember

- Terraform workspaces separate **state** for the same configuration (classically dev/staging/prod of identical infra shape) — they do not switch providers or which cloud is targeted.
- Multiple `provider` blocks in one configuration are what actually let a single Terraform codebase manage resources across AWS, GCP, Azure, etc., in one `plan`/`apply`.
- Workspaces and multi-provider configs address different axes (environment vs. cloud) and can be combined, but many teams prefer separate root modules per environment over heavy `terraform.workspace ==` conditionals for readability.
- You need only one Terraform state backend (with locking) even when the managed resources span multiple clouds — the backend's cloud is an independent choice from the providers being managed.
- `required_providers` in the `terraform` block must declare every provider (aws, google, etc.) the configuration uses, each pinned to a version constraint, exactly like a single-cloud setup.

## Common Mistakes

- Trying to use one workspace per cloud provider (`terraform workspace new aws`, `terraform workspace new gcp`) expecting it to isolate which provider's resources get created — workspaces don't do this; you'd still need conditional logic per resource, which doesn't scale.
- Scattering `terraform.workspace == "x"` conditionals throughout resource blocks to fake per-cloud or per-environment behavior, producing a configuration that's hard to read and easy to misconfigure (forgetting a conditional on a new resource).
- Standing up a separate Terraform state backend per cloud provider for what's logically one environment's infrastructure, fragmenting state and making a single `plan` unable to show the full picture of what would change.
- Forgetting to pin provider versions (`required_providers` with version constraints) for every cloud used — this is just as important in a multi-cloud config as a single-cloud one, and a provider version drift bug is harder to diagnose when there are more providers in play.
- Assuming multi-provider Terraform automatically means safe cross-cloud dependency ordering — Terraform's dependency graph handles this fine via implicit/explicit `depends_on`, but it's easy to introduce a real cross-cloud race (e.g., a GCP resource referencing an AWS-side value that isn't ready yet) if data flow between the two isn't modeled explicitly.
