# Day 47 — Cheatsheet: Terraform on AWS Full Project

## Backend (S3 + DynamoDB)

```hcl
terraform {
  backend "s3" {
    bucket         = "my-org-terraform-state"
    key            = "prod/eks-cluster/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

```bash
terraform init
terraform init -reconfigure     # after changing backend block
terraform init -migrate-state   # moving from local to remote backend
terraform force-unlock <lock-id>  # only if you're SURE no one else is applying
```

## VPC module (terraform-aws-modules)

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "prod-vpc"
  cidr = "10.0.0.0/16"
  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  enable_nat_gateway = true
  single_nat_gateway = true   # cost tradeoff: false = one NAT per AZ (HA, pricier)

  public_subnet_tags  = { "kubernetes.io/cluster/prod" = "shared", "kubernetes.io/role/elb" = "1" }
  private_subnet_tags = { "kubernetes.io/cluster/prod" = "shared", "kubernetes.io/role/internal-elb" = "1" }
}
```

## EKS module

```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = "prod"
  cluster_version = "1.30"
  vpc_id          = module.vpc.vpc_id
  subnet_ids      = module.vpc.private_subnets

  eks_managed_node_groups = {
    default = { instance_types = ["m6i.large"], min_size = 2, max_size = 6, desired_size = 3 }
  }
}
```

## IRSA

```hcl
module "irsa_example" {
  source  = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"
  version = "~> 5.0"

  role_name                     = "my-app-role"
  attach_load_balancer_controller_policy = true

  oidc_providers = {
    main = {
      provider_arn               = module.eks.oidc_provider_arn
      namespace_service_accounts = ["kube-system:aws-load-balancer-controller"]
    }
  }
}
```

```yaml
# ServiceAccount annotation the pod needs
metadata:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/my-app-role
```

## Outputs & cross-state reference

```hcl
output "cluster_name"      { value = module.eks.cluster_name }
output "cluster_endpoint"  { value = module.eks.cluster_endpoint }
output "oidc_provider_arn" { value = module.eks.oidc_provider_arn }
output "db_password"       { value = random_password.db.result; sensitive = true }
```

```hcl
data "terraform_remote_state" "vpc" {
  backend = "s3"
  config = { bucket = "my-org-terraform-state", key = "prod/vpc/terraform.tfstate", region = "us-east-1" }
}
# data.terraform_remote_state.vpc.outputs.vpc_id

terraform output                 # human-readable
terraform output -json           # for scripts/CI
terraform output -raw cluster_name
```

## Atlantis (`atlantis.yaml`)

```yaml
version: 3
projects:
- name: prod-eks
  dir: environments/prod
  workspace: default
  autoplan:
    when_modified: ["*.tf", "*.tfvars"]
  apply_requirements: [approved, mergeable]
```
PR comments: `atlantis plan`, `atlantis apply`, `atlantis unlock`.

## GitHub Actions apply flow (skeleton)

```yaml
jobs:
  plan:
    if: github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - run: terraform -chdir=environments/prod init
      - run: terraform -chdir=environments/prod plan -out=tfplan
  apply:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    environment: production   # requires manual reviewer approval
    steps:
      - run: terraform -chdir=environments/prod apply -auto-approve
```

## eksctl comparison (mentioned as "compare" in today's tools)

```bash
eksctl create cluster --name dev --version 1.30 --nodegroup-name ng-1 --nodes 3
```
`eksctl`: fast, imperative, great for throwaway/dev clusters. Terraform: declarative, versioned, integrates with the rest of your IaC and state — the standard for anything long-lived/production.
