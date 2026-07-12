# Day 34 — Resources: AWS Compute (EKS, ECS, Lambda)

## Primary (assigned)

- **AWS EKS Best Practices Guide** (aws.github.io/aws-eks-best-practices, free) — the assigned resource. Read the "Cluster Autoscaling" and "Fargate" sections directly, since they cover exactly the compute-option tradeoffs in file 1, written by the EKS service team itself.

## Deepen your understanding

- **AWS — "Amazon EKS nodes" documentation** (docs.aws.amazon.com/eks/latest/userguide/eks-compute.html) — official comparison of self-managed nodes, managed node groups, and Fargate.
- **AWS — "Choosing an AWS container service"** (aws.amazon.com/containers/services) — AWS's own decision-guide comparing ECS, EKS, App Runner, and Fargate side by side; directly useful for file 2's ECS vs. EKS discussion.
- **AWS Lambda documentation — "Understanding Lambda function scaling"** and **"Configuring provisioned concurrency"** — the authoritative reference on cold starts, concurrency, and provisioned concurrency mechanics.
- **AWS — "Amazon EventBridge documentation"**, especially the "Event patterns" reference page — covers exactly how rule matching works for the Lambda/EventBridge comparison in file 3.

## Reference / lookup

- **AWS SAM CLI documentation** (docs.aws.amazon.com/serverless-application-model) — command reference for `sam init`/`build`/`deploy`/`logs` used throughout today's lab.
- **`eksctl` documentation** (eksctl.io) — full CLI reference for cluster, nodegroup, and Fargate profile creation.

## Practice

- **AWS Skill Builder — "Running Containers on Amazon Elastic Kubernetes Service (Amazon EKS)" (free digital course)** — structured, hands-on-adjacent practice reinforcing today's EKS compute-option material.
