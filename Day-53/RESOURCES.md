# Day 53 — Resources: Phase 1 Project — Full Stack IaC

## Primary (assigned)

- **Your own Phase 1 notes** (Days 22–52 of this repo) — the assigned starting point; this project is explicitly a synthesis of everything already covered (Terraform, EKS, RDS/ElastiCache, Helm, ArgoCD, secrets management, autoscaling). Re-read your Day-49 through Day-52 notes alongside the earlier Terraform/EKS/Helm days before starting.

## Deepen your understanding

- **terraform-aws-modules/eks/aws and terraform-aws-modules/vpc/aws GitHub repos** — the actual, production-grade community modules referenced in this project's code samples; reading their full variable lists reveals many production knobs (IRSA helpers, add-on management) beyond what's shown here.
- **ArgoCD documentation — "App of Apps" pattern** — the canonical pattern for bootstrapping an entire platform (app + supporting controllers) from a single Git source, directly relevant to structuring this project's repo.
- **AWS documentation — IAM Roles for Service Accounts (IRSA)** — the exact mechanism behind every in-cluster controller (Load Balancer Controller, External Secrets, Karpenter) getting scoped AWS permissions without static credentials; worth understanding at the OIDC-federation level, not just the YAML.
- **Karpenter documentation — NodePool and consolidation concepts** — goes deeper into consolidation policies, disruption budgets, and how Karpenter's bin-packing decision-making actually works, beyond the summary in this project's notes.

## Reference / lookup

- `terraform providers schema -json` and `terraform state list` — inspect exactly what a module creates without leaving the CLI.
- **External Secrets Operator documentation — AWS Secrets Manager provider** — full auth method reference (IRSA, static credentials, EKS Pod Identity) and every field on `SecretStore`/`ExternalSecret`.
- **KEDA scalers reference** (keda.sh/docs/scalers) — the full catalog of ~70 event source scalers beyond AWS SQS, useful for recognizing which scaler fits a given architecture in an interview.

## Practice

- **A disposable AWS sandbox account** — build this project for real; a from-memory description of an architecture is a weaker interview answer than one grounded in something you actually stood up and tore down yourself.
- **KillerCoda / A Cloud Guru EKS scenario labs** — for practicing individual pieces (ArgoCD sync behavior, Karpenter provisioning) in isolation before assembling the full stack, if the full AWS build feels too large to attempt in one sitting.
