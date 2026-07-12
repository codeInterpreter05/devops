# Day 47 — Terraform on AWS: Full EKS Cluster Architecture & Module Registry

**Phase:** 1 – Core DevOps | **Week:** W7 | **Domain:** Terraform | **Flag:** 📌 Reference

## Brief

Everything you've learned about EKS piecemeal (VPC design, IAM roles for service accounts, load balancer integration) comes together on this day as one real, production-shaped Terraform project. This is the single most common "show me you can actually build infrastructure" interview and take-home exercise in DevOps hiring — "walk me through your Terraform project structure for a multi-environment EKS setup" (this day's interview question) is asked because the answer reveals whether you understand module boundaries, state isolation, and reusability, not just whether you can write HCL that happens to apply successfully once.

This day is split into three files:

1. **This file** — the full EKS architecture in Terraform (VPC + EKS + node groups + IRSA + ALB) and the module registry.
2. **[02-README-Remote-State-Locking.md](02-README-Remote-State-Locking.md)** — remote state storage and locking.
3. **[03-README-CI-CD-Apply-Flow.md](03-README-CI-CD-Apply-Flow.md)** — the Atlantis/GitHub Actions apply flow and outputs.

## The full stack, piece by piece

A production EKS cluster in Terraform is really **four layers composed together**, and understanding the dependency direction between them is the core of the architecture question:

```
VPC (subnets, NAT, route tables)
  -> EKS cluster (control plane, OIDC provider)
       -> Node groups (managed node groups and/or Fargate profiles)
       -> IRSA (IAM Roles for Service Accounts, depends on the OIDC provider)
            -> aws-load-balancer-controller (deployed via Helm, needs an IRSA role)
                 -> ALB (created dynamically by the controller via Ingress/Service objects, not by Terraform directly)
```

**VPC layer**: almost nobody hand-rolls this — the community `terraform-aws-modules/vpc/aws` module is the de facto standard, handling public/private subnet splitting across AZs, NAT Gateway placement (one per AZ for HA, or a single shared one to save cost — a real tradeoff to state explicitly), and the specific subnet tags EKS and the AWS Load Balancer Controller require for auto-discovery:
```hcl
public_subnet_tags = {
  "kubernetes.io/cluster/${var.cluster_name}" = "shared"
  "kubernetes.io/role/elb"                    = "1"
}
private_subnet_tags = {
  "kubernetes.io/cluster/${var.cluster_name}" = "shared"
  "kubernetes.io/role/internal-elb"            = "1"
}
```
Missing these tags is the single most common reason `aws-load-balancer-controller` can't auto-discover subnets for a new ALB — it's a silent failure that only shows up as a stuck `Ingress` with no address.

**EKS cluster layer**: `terraform-aws-modules/eks/aws` (this day's named "Best Resource," Anton Babenko's module) wraps the `aws_eks_cluster` resource plus the OIDC provider needed for IRSA, cluster security group rules, and (in recent versions) the new **access entries** API for cluster auth instead of the legacy `aws-auth` ConfigMap:
```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = var.cluster_name
  cluster_version = "1.30"
  vpc_id          = module.vpc.vpc_id
  subnet_ids      = module.vpc.private_subnets

  enable_cluster_creator_admin_permissions = true

  eks_managed_node_groups = {
    default = {
      instance_types = ["m6i.large"]
      min_size       = 2
      max_size       = 6
      desired_size   = 3
    }
  }
}
```

**IRSA (IAM Roles for Service Accounts)**: this is the mechanism that lets a specific Kubernetes ServiceAccount assume a specific IAM role — no static AWS credentials in a pod, no over-broad node-level IAM role granting every pod on that node the same permissions. Mechanically: the EKS cluster's **OIDC provider** (created as part of the cluster) is trusted by an IAM role's trust policy, scoped to a specific namespace+ServiceAccount via a condition on the OIDC subject claim:
```hcl
module "lb_controller_irsa" {
  source  = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"
  version = "~> 5.0"

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
The pod then just needs an annotation on its ServiceAccount (`eks.amazonaws.com/role-arn: <role-arn>`) — the EKS Pod Identity webhook injects the right environment variables/projected token automatically, no application code changes needed to pick up temporary, scoped AWS credentials.

**ALB / aws-load-balancer-controller**: note this is deliberately **not** provisioned by Terraform directly — the controller is installed via Helm (often itself invoked from Terraform's `helm_provider`, or more cleanly from a separate CD step — see file 03) and then creates/manages actual ALB/NLB resources dynamically in response to `Ingress`/`Service type=LoadBalancer` objects applied via `kubectl`/Helm/GitOps. Terraform's job stops at "give the controller the IAM permissions and VPC tags it needs"; the controller's job is the ongoing reconciliation of load balancers from Kubernetes objects — a clean division of responsibility between infrastructure-as-code and cluster-native reconciliation, worth stating explicitly in an interview.

## Terraform Module Registry — build vs. reuse

The public Terraform Registry (registry.terraform.io) hosts community and HashiCorp-verified modules. For AWS/EKS specifically, `terraform-aws-modules` (Anton Babenko's org) is the effective standard for VPC, EKS, IAM/IRSA, RDS, security groups, and more.

**Why default to a registry module instead of hand-writing the resource blocks:**
- These modules encode dozens of edge cases and AWS API quirks (e.g., correct subnet tagging, correct dependency ordering between OIDC provider creation and IRSA role creation) that took the community years to shake out — writing your own from scratch means re-discovering those the hard way, usually during an incident.
- **Pin versions explicitly** (`version = "~> 20.0"`) — registry modules do introduce breaking changes across major versions (the EKS module has had several major rewrites as EKS itself evolved, e.g., the v19→v20 jump changing node group defaults) — an un-pinned module reference is a landmine for "it worked yesterday, why did `terraform plan` suddenly want to replace my cluster today."
- **When to write your own module instead**: once you have 2+ real consumers of the same reusable pattern (e.g., every microservice team needs "a namespace + IRSA role + KafkaTopic" as one bundle) — that's an internal module candidate, following the same principle as "don't abstract until you have a second real use case" from general software engineering.

**Local module composition** — the practical pattern this day's hands-on activity assumes — is wrapping registry modules inside your own thin `modules/eks-cluster` local module that fixes your organization's specific defaults (tagging standards, standard node group shapes, standard IRSA roles every cluster needs), so a new environment is `module "prod" { source = "./modules/eks-cluster" ... }` with only the environment-specific variables exposed — not a 500-line copy-pasted root module per environment.

## Points to Remember

- Layer dependency direction: VPC → EKS cluster (+ OIDC provider) → node groups + IRSA roles → controllers/workloads that consume those IRSA roles. Get this order backwards and Terraform's implicit dependency graph (or your explicit `depends_on`) will fail to apply cleanly.
- IRSA scopes IAM permissions to a specific namespace+ServiceAccount via the OIDC provider's trust policy — avoiding both static credentials and overly broad node-level IAM permissions.
- Public subnet tags (`kubernetes.io/role/elb`) and private subnet tags (`kubernetes.io/role/internal-elb`) are required for the AWS Load Balancer Controller's auto-discovery — a very common silent-failure point if missed.
- Terraform provisions the IAM/VPC scaffolding the load balancer controller needs; the controller itself (via Kubernetes objects) provisions and manages actual ALB/NLB resources — not Terraform directly.
- Always pin registry module versions explicitly; major version bumps in `terraform-aws-modules/eks/aws` have historically changed defaults significantly enough to cause unwanted resource replacement.

## Common Mistakes

- Forgetting the EKS-required subnet tags in a hand-rolled VPC (or an older/customized VPC module config), leading to Ingress objects stuck without an ALB address and no obvious error message pointing at the real cause.
- Hardcoding a broad node-instance-level IAM role with permissions every pod on that node inherits, instead of using IRSA to scope permissions per-ServiceAccount — a real security regression that's easy to fall into if IRSA feels like "extra setup."
- Using an un-pinned or loosely pinned (`>= 19.0`) module version in a shared root module, then getting a surprise plan diff wanting to replace node groups or the cluster itself after a routine `terraform init -upgrade`.
- Copy-pasting an entire root module per environment (dev/staging/prod) instead of parameterizing one module with environment-specific `tfvars` — leads to configuration drift between environments that nobody notices until an incident only reproduces in prod.
- Trying to manage ALB/Target Group resources directly in Terraform *and* running the AWS Load Balancer Controller for the same Ingress — a classic "two systems fighting over the same resource" conflict that causes flapping/unexpected reverts.
