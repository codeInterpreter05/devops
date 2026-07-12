# Day 77 — Cheatsheet: Multi-cloud CI/CD

## One build, parallel multi-cloud deploy jobs (GitHub Actions)

```yaml
jobs:
  build:
    outputs: { digest: ${{ steps.build.outputs.digest }} }
    steps:
      - id: build
        uses: docker/build-push-action@v5
        with: { push: true, tags: registry/app:${{ github.sha }} }

  deploy-aws:
    needs: build
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with: { role-to-assume: arn:aws:iam::111:role/gha-deploy, aws-region: us-east-1 }
      - run: aws lambda update-function-code --function-name app --image-uri registry/app@${{ needs.build.outputs.digest }}

  deploy-gcp:
    needs: build
    steps:
      - uses: google-github-actions/auth@v2
        with: { workload_identity_provider: projects/123/.../providers/gha, service_account: gha@proj.iam.gserviceaccount.com }
      - run: gcloud run deploy app --image=registry/app@${{ needs.build.outputs.digest }} --region=us-central1
```

## Terraform: multiple providers, one config

```hcl
terraform {
  required_providers {
    aws    = { source = "hashicorp/aws" }
    google = { source = "hashicorp/google" }
  }
  backend "s3" {
    bucket = "myorg-tfstate"
    key    = "multi-cloud/terraform.tfstate"
    dynamodb_table = "tf-lock"
  }
}

provider "aws"    { region  = "us-east-1" }
provider "google" { project = "my-project"; region = "us-central1" }

resource "aws_lambda_function" "app" { /* ... */ }
resource "google_cloud_run_v2_service" "app" { /* ... */ }
```

## Terraform workspaces (environment axis, NOT cloud axis)

```bash
terraform workspace new staging
terraform workspace new production
terraform workspace list
terraform workspace select staging
terraform workspace show
terraform apply -var-file=staging.tfvars
```

```hcl
# workspace controls environment sizing, not which provider is used
count = terraform.workspace == "production" ? 3 : 1
```

## OIDC federation (no static cloud credentials)

```yaml
# AWS
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::ACCOUNT:role/gha-deploy
    aws-region: us-east-1

# GCP
- uses: google-github-actions/auth@v2
  with:
    workload_identity_provider: projects/PROJECT_NUM/locations/global/workloadIdentityPools/POOL/providers/PROVIDER
    service_account: SA_EMAIL
```

## Multi-cloud cost/justification quick-check

```
Justified:
  - regulatory / data-residency mandate
  - M&A transitional state
  - narrow best-of-breed service for one bounded workload
  - catastrophic-risk hedge for a small set of truly critical systems

Weak justification (commonly cited, rarely pays off):
  - "avoid vendor lock-in" (blanket policy)
  - "negotiating leverage" (rarely acted on in practice)

Real ongoing costs:
  - ~2x on-call cognitive load (different IAM/network/failure models per cloud)
  - cross-cloud egress fees
  - forfeited volume/committed-use discounts (spend split across providers)
  - duplicated tooling/observability configuration per cloud
```

## Repatriation signal checklist

```
[ ] Workload has stable, predictable, high-volume usage (not bursty/elastic)
[ ] Steady-state cloud bill exceeds owned/colocated cost over a multi-year horizon
[ ] Workload is storage- or bandwidth-heavy (classic repatriation candidates)
-> if most boxes checked: partial repatriation may be worth modeling
-> elastic/variable/new workloads: usually stay on public cloud
```
