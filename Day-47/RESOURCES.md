# Day 47 — Resources: Terraform on AWS Full Project

## Primary (assigned)

- **Anton Babenko's EKS Terraform module** (github.com/terraform-aws-modules/terraform-aws-eks) — free, the assigned starting point. The de facto standard module for EKS-on-Terraform; the README and `examples/` directory cover node groups, Fargate profiles, and IRSA patterns directly from the source.

## Deepen your understanding

- **terraform-aws-modules/vpc** (github.com/terraform-aws-modules/terraform-aws-vpc) — the companion VPC module, including the exact subnet tagging behavior EKS and the load balancer controller depend on.
- **"IAM roles for service accounts" (AWS EKS docs)** — the authoritative mechanical explanation of the OIDC trust relationship underlying IRSA.
- **Atlantis documentation** (runatlantis.io/docs) — covers `atlantis.yaml` project configuration, locking behavior, and custom workflows in full.
- **HashiCorp "Terraform at Scale" / Terraform Cloud documentation** — worth skimming even if you use plain S3 backend + Atlantis/Actions, since it frames the same state-isolation and review-gate problems this day's notes cover, from HashiCorp's own recommended-practices angle.

## Reference / lookup

- **Terraform Registry** (registry.terraform.io) — search here before writing any AWS resource block by hand; check module popularity, maintenance recency, and pinned version compatibility.
- **`terraform state` subcommand reference** (`state list`, `state show`, `state mv`, `state rm`) — needed the moment you have to safely restructure modules without destroying/recreating real resources.
- **AWS Load Balancer Controller documentation** (kubernetes-sigs.github.io/aws-load-balancer-controller) — the controller side of the ALB/Ingress mechanics referenced in file 01.

## Practice

- **A from-scratch rebuild of this day's lab in a second AWS region/account** — repeating VPC+EKS+IRSA+ALB from memory (checking the module docs only when stuck) is the fastest way to actually internalize the dependency ordering rather than just having copy-pasted it once.
- **Convert your Lab 4 GitHub Actions setup to Atlantis** (the stretch challenge) — direct hands-on comparison of the two CI/CD apply flows is more instructive than reading about the tradeoff.
