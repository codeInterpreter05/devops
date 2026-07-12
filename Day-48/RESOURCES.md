# Day 48 — Resources: EKS Deep Dive

## Primary (assigned)

- **EKS Best Practices Guide** (aws.github.io/aws-eks-best-practices) — free, the assigned starting point. Dedicated sections on networking, cluster autoscaling, and security cover essentially everything in this day's four files, written by the AWS EKS team directly.

## Deepen your understanding

- **"Amazon VPC CNI plugin for Kubernetes" GitHub repo** (github.com/aws/amazon-vpc-cni-k8s) — the README and `docs/` directory explain `ipamd`, prefix delegation, and custom networking in full mechanical detail, straight from the source.
- **Karpenter documentation** (karpenter.sh) — the "Concepts" and "NodePools" pages cover consolidation, disruption budgets, and provisioning behavior in more depth than this day's notes.
- **"Introducing Amazon EKS Access Entries" (AWS blog post)** — the original announcement post walks through the migration path from `aws-auth` with concrete before/after examples.
- **EKS Anywhere documentation** (anywhere.eks.amazonaws.com) — covers supported providers (vSphere, bare metal, Snow, Docker, CloudStack) and the exact relationship to EKS Distro.

## Reference / lookup

- **`amazon-vpc-cni-k8s` "eni-max-pods.txt"** — the definitive per-instance-type max-pod reference table used to compute EKS node pod density limits.
- **AWS Load Balancer Controller documentation — "Ingress annotations"** — reference for `alb.ingress.kubernetes.io/target-type` (instance vs. ip) and other IP-target-mode-relevant settings.
- **Cluster Autoscaler GitHub repo — AWS cloud provider docs** — for anyone maintaining a CA setup rather than migrating away, the exact IAM permissions and ASG tagging requirements are documented here.

## Practice

- **Karpenter "Getting Started with Karpenter" tutorial** (karpenter.sh/docs/getting-started) — the official guided walkthrough this day's lab is modeled on, useful as a second reference if the lab's steps hit an environment-specific snag.
- **killercoda EKS-focused scenarios** — several free browser-based scenarios cover VPC CNI limits and access control hands-on without needing your own AWS account.
