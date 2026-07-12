# Day 53 — Phase 1 Project: Infrastructure Layer in Terraform

**Phase:** 1 – Core DevOps | **Week:** W8 | **Domain:** Review | **Flag:** 📌 Milestone project

## Brief

Today is a synthesis day, not a new-topic day: you're assembling almost every AWS + Terraform + Kubernetes concept from Phase 1 into one coherent, production-shaped environment — VPC, EKS, RDS, ElastiCache, and an ALB, all defined as code. The point isn't to learn new mechanics; it's to prove (to an interviewer, and to yourself) that you can reason about how these pieces fit together end-to-end, not just operate each one in isolation. This is exactly the shape of today's interview question: "walk me through the complete architecture of a production EKS environment you've built" — the strongest answers are the ones that explain *why* each layer is structured the way it is, not just *what* was deployed.

This day is split into three files:

1. **This file** — the infrastructure layer: VPC, EKS, RDS, ElastiCache, ALB, all in Terraform.
2. **[02-README-Deployment-And-Secrets.md](02-README-Deployment-And-Secrets.md)** — deploying the app with Helm + ArgoCD, and secrets via External Secrets + AWS Secrets Manager.
3. **[03-README-Autoscaling-KEDA-Karpenter.md](03-README-Autoscaling-KEDA-Karpenter.md)** — auto-scaling with KEDA and Karpenter.

## Why this architecture shape, and in this order

A production EKS environment is built in **layers with dependencies flowing one direction**: networking first, because everything else (EKS control plane, RDS, ElastiCache, ALB) needs to live inside a VPC with correctly sized subnets and routing already in place; compute (EKS) next, since workloads need somewhere to run before they can talk to data stores; data stores (RDS, ElastiCache) alongside or just after compute, since the app needs them reachable at deploy time; and the ALB last, as the thing that exposes the finished stack to the outside world. Terraform module structure should mirror this dependency order — it's both a design clarity win and a blast-radius control: a mistake in the ALB module shouldn't be able to touch VPC routing.

## VPC design for EKS

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "prod-eks-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  enable_nat_gateway   = true
  single_nat_gateway   = false   # one NAT per AZ in prod — avoids cross-AZ data transfer + single point of failure
  enable_dns_hostnames = true

  # Required tags for the EKS/AWS Load Balancer Controller to auto-discover subnets
  public_subnet_tags = {
    "kubernetes.io/role/elb"                      = "1"
    "kubernetes.io/cluster/prod-eks-cluster"       = "shared"
  }
  private_subnet_tags = {
    "kubernetes.io/role/internal-elb"              = "1"
    "kubernetes.io/cluster/prod-eks-cluster"       = "shared"
  }
}
```

**Why three AZs, not two**: EKS control plane and most managed data stores (RDS Multi-AZ, ElastiCache replication groups) want at least 2 AZs for HA, but 3 is the practical standard because it lets you tolerate a full AZ outage while still having quorum/majority for anything using leader election, and it spreads NAT Gateway cost/traffic more evenly. **Why private subnets for workloads and data stores**: nodes, pods, RDS, and ElastiCache should have no direct route from the public internet — they reach the internet outbound (for pulling images, calling external APIs) via a NAT Gateway, and are reached inbound only through the ALB in the public subnet. **Why the specific subnet tags**: the AWS Load Balancer Controller and the Kubernetes cloud provider auto-discover which subnets to place load balancers in and which are "internal" by reading these exact tag keys — get the tags wrong and ALB provisioning silently fails to find subnets, or picks the wrong ones.

## EKS cluster and node groups

```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = "prod-eks-cluster"
  cluster_version = "1.29"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets   # nodes go in private subnets

  cluster_endpoint_public_access  = true    # kubectl access from outside the VPC
  cluster_endpoint_private_access = true    # in-VPC access without traversing the internet

  eks_managed_node_groups = {
    general = {
      instance_types = ["m5.large"]
      min_size       = 2
      max_size       = 6
      desired_size   = 3
    }
  }

  cluster_addons = {
    vpc-cni    = { most_recent = true }
    coredns    = { most_recent = true }
    kube-proxy = { most_recent = true }
    aws-ebs-csi-driver = { most_recent = true }
  }
}
```

**Why both public and private endpoint access, not just one**: private-only means CI/CD runners and engineers must be inside the VPC (via VPN/bastion) to reach the API server — more secure but more operational friction; public-only means the API server is internet-reachable (locked down via IAM + the cluster's own auth, but still a larger attack surface). Most real production setups enable both and additionally restrict public access to specific CIDR ranges (office IPs, VPN egress) via `cluster_endpoint_public_access_cidrs`, getting convenient access without leaving the endpoint open to the whole internet.

## RDS and ElastiCache placement

```hcl
resource "aws_db_subnet_group" "main" {
  name       = "prod-db-subnet-group"
  subnet_ids = module.vpc.private_subnets
}

resource "aws_db_instance" "postgres" {
  identifier             = "prod-app-db"
  engine                 = "postgres"
  engine_version         = "16.3"
  instance_class         = "db.r6g.large"
  allocated_storage      = 100
  multi_az               = true                 # sync standby in another AZ, automatic failover
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]
  manage_master_user_password = true             # RDS-managed rotation via Secrets Manager, no password in state/code
  backup_retention_period = 7
  deletion_protection     = true
}

resource "aws_elasticache_replication_group" "redis" {
  replication_group_id       = "prod-app-cache"
  engine                     = "redis"
  node_type                  = "cache.r6g.large"
  num_cache_clusters         = 2                 # primary + 1 replica
  automatic_failover_enabled = true
  subnet_group_name          = aws_elasticache_subnet_group.main.name
  security_group_ids         = [aws_security_group.redis.id]
  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
}
```

**`manage_master_user_password = true`** is the specific detail worth calling out in an interview: it tells RDS to generate and store the master password directly in AWS Secrets Manager, rotated automatically, so the password never appears in Terraform state or version control as plaintext — a meaningfully better default than the older pattern of generating a `random_password` resource and passing it in as a variable (which does land in state, encrypted at rest by the backend but still a real exposure surface if state access controls are loose). Security group rules for both RDS and Redis should allow ingress **only from the EKS node/pod security group**, never `0.0.0.0/0` — this is the kind of detail that separates a real production config from a tutorial one.

## ALB via the AWS Load Balancer Controller

The ALB itself typically isn't hand-authored as a raw `aws_lb` Terraform resource for EKS workloads — it's provisioned **dynamically by the AWS Load Balancer Controller** (installed via Helm, itself often bootstrapped by Terraform) reading Kubernetes `Ingress` resources annotated for ALB. Terraform's job is to set up the *prerequisites*: the IAM role (via IRSA — IAM Roles for Service Accounts) the controller needs to create/manage ALBs on your behalf, and installing the controller itself.

```hcl
module "lb_controller_irsa" {
  source  = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"
  role_name = "aws-load-balancer-controller"
  attach_load_balancer_controller_policy = true
  oidc_providers = {
    main = {
      provider_arn               = module.eks.oidc_provider_arn
      namespace_service_accounts = ["kube-system:aws-load-balancer-controller"]
    }
  }
}
```
Then a Kubernetes `Ingress` object (deployed via Helm/ArgoCD, covered in the next file) with `alb.ingress.kubernetes.io/*` annotations is what actually causes the controller to create the ALB, target groups, and listener rules — Terraform sets the stage, Kubernetes manifests trigger the actual ALB creation. This is a commonly misunderstood boundary: **Terraform doesn't own the ALB lifecycle for EKS ingress traffic; the in-cluster controller does**, reacting to Kubernetes objects.

## Points to Remember

- Layer order: VPC → EKS (+ IRSA for controllers) → RDS/ElastiCache → ALB (via in-cluster controller reacting to Ingress objects) — each layer depends on the one before it.
- Workloads and data stores live in private subnets with NAT Gateway egress; only the ALB sits in public subnets — nothing reaches RDS/ElastiCache/nodes directly from the internet.
- Specific subnet tags (`kubernetes.io/role/elb`, `kubernetes.io/cluster/<name>`) are how the Load Balancer Controller and Kubernetes cloud provider auto-discover subnets — get them wrong and provisioning silently misbehaves.
- `manage_master_user_password = true` on RDS avoids ever having the master password pass through Terraform state or code as plaintext, by delegating storage/rotation to Secrets Manager directly.
- The ALB for EKS ingress is provisioned by the in-cluster AWS Load Balancer Controller reacting to `Ingress` objects, not directly authored as a Terraform `aws_lb` resource — Terraform's job is IAM/IRSA plumbing and installing the controller.

## Common Mistakes

- Using a single NAT Gateway for a multi-AZ VPC to save cost, creating a single point of failure and unnecessary cross-AZ data transfer charges for a "production" environment.
- Forgetting the exact subnet tag keys/values the Load Balancer Controller expects, then spending hours debugging why an Ingress never gets an ALB address.
- Opening RDS/ElastiCache security groups to `0.0.0.0/0` "temporarily" during initial setup and forgetting to lock it down to the node/pod security group before considering the environment production-ready.
- Managing the master DB password via a Terraform `random_password` + variable pattern out of habit, when `manage_master_user_password` (RDS-native Secrets Manager integration) is both simpler and more secure.
- Trying to hand-author the ALB as a raw Terraform `aws_lb` resource for Kubernetes ingress traffic instead of letting the AWS Load Balancer Controller manage it — this fights against how Kubernetes-native ALB provisioning is designed to work and loses automatic target-group registration as pods scale.
