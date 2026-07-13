# Day 113 — Crossplane & Self-Service Infra: Crossplane vs Terraform

**Phase:** 4 – Advanced/Specialization | **Week:** W19 | **Domain:** Platform Eng | **Flag:** —

## Brief

"How does Crossplane differ from Terraform? What problem does it solve for platform teams?" is today's flagged interview question, and it's a genuinely important distinction to get right — the two tools solve overlapping problems (provisioning cloud infrastructure declaratively) with fundamentally different execution and operating models, and conflating them ("Crossplane is just Terraform on Kubernetes") is the single most common wrong answer.

## The core architectural difference: continuous reconciliation vs. run-based apply

**Terraform** is fundamentally a **CLI tool that runs a plan/apply cycle on demand** (or on a CI schedule/trigger): you run `terraform plan`, review a diff, run `terraform apply`, and Terraform makes the API calls needed to converge real infrastructure to match your `.tf` files, using state recorded in a **state file** (local, or remote in S3/Terraform Cloud/etc.). Between runs, nothing is watching — if someone manually changes a resource in the AWS console, Terraform doesn't know or react until the *next* `plan`/`apply` is explicitly run.

**Crossplane** runs as **long-lived controllers inside a Kubernetes cluster**, continuously reconciling in a loop — the same control-loop model as every native Kubernetes controller. There's no separate "state file" concept; the current desired state is just whatever's in the CRD objects in the cluster (itself stored in etcd, the same place all Kubernetes state lives), and drift is detected and corrected continuously, not just when someone remembers to run a command.

This distinction is *the* correct interview answer: **Terraform is push-based/on-demand (you trigger convergence); Crossplane is control-loop-based/continuous (convergence is always running).**

## Practical consequences of that difference

| Dimension | Terraform | Crossplane |
|---|---|---|
| Execution model | Run plan/apply on demand or in CI | Continuously running controllers in-cluster |
| Drift handling | Detected/fixed only on next `plan`/`apply` | Detected/fixed automatically, continuously |
| State storage | Separate state file (local/remote backend) | Kubernetes API objects (etcd) — no separate state store |
| Consumer-facing API | HCL files, usually behind a CI pipeline/PR | Kubernetes CRDs — `kubectl apply`, GitOps-native |
| Self-service story | Needs extra tooling (Atlantis, Terraform Cloud, custom wrappers) to be safely self-service | Native to the model — Claims/XRDs *are* the self-service layer, RBAC-scoped |
| Extending to new resource types | Provider release cadence + HCL schema | Provider release cadence + CRD schema (similar) |
| Multi-cloud composition in one call | Possible via modules, but still a single apply run | Native — a Composition can mix AWS + GCP + Helm resources in one Composite Resource |
| GitOps fit | Needs a CI step to actually run `apply`; GitOps tools don't natively drive HCL applies | Fits directly into ArgoCD/Flux — CRDs are just Kubernetes objects like anything else |

## What problem this actually solves for platform teams

The reason platform teams reach for Crossplane specifically (rather than "just wrap Terraform in a nicer UI") is that **self-service infrastructure needs an API developers can call safely, repeatedly, without a human reviewing a Terraform plan every time** — and Kubernetes' CRD, RBAC, and admission-control model already *is* exactly that kind of governed, multi-tenant API surface. Building the equivalent safety/guardrail layer on top of raw Terraform (who can run `apply`, with what variables, against what state, with what approval) means building a custom platform from scratch — this is literally what tools like Atlantis, Spacelift, and Terraform Cloud's team/policy features exist to bolt on. Crossplane gets that governance model "for free" from Kubernetes primitives that most platform teams are already running and already understand (RBAC, namespaces, admission webhooks, GitOps reconciliation via ArgoCD/Flux).

The honest tradeoff: Terraform's provider ecosystem is broader and more mature (more resource types supported, more battle-tested modules, a much larger community module registry), and for teams who don't already run Kubernetes as their control plane, standing up Crossplane just to provision infra is significant added operational surface. Many real platform teams therefore run **both**: Terraform (often behind Atlantis/Terraform Cloud) for foundational, rarely-changed infra (VPCs, IAM, org-wide networking) managed by the platform team itself, and Crossplane-exposed self-service Claims for the frequently-requested, per-application resources (databases, buckets, queues) that app teams need on demand.

## Points to Remember

- The core distinction: Terraform = on-demand plan/apply with a state file; Crossplane = continuous in-cluster reconciliation with Kubernetes objects as the source of truth, no separate state file.
- Drift correction is automatic and continuous in Crossplane; in Terraform it only happens the next time someone runs `plan`/`apply`.
- Crossplane's real value-add for platform teams is that Kubernetes' RBAC/CRD/admission-control model gives you a governed self-service API "for free" — building the same safe self-service story on raw Terraform requires extra tooling (Atlantis, Terraform Cloud, custom wrappers).
- Terraform's provider ecosystem and module registry are broader and more mature; Crossplane's advantage is the native fit with Kubernetes-centric platforms and GitOps tooling, not raw resource-type coverage.
- Many organizations run both — Terraform for foundational/rarely-changed infra, Crossplane Claims for frequent, per-team self-service resources — this is a valid and common answer, not an either/or.

## Common Mistakes

- Answering "Crossplane is Terraform for Kubernetes" in an interview — this misses the actual architectural distinction (continuous control loop vs. on-demand run) that's the point of the question.
- Assuming Crossplane eliminates the need for any CI/CD pipeline — Claims/Compositions still need to be authored, reviewed, and typically GitOps-deployed (ArgoCD/Flux watching a repo of Claim YAML) — the pipeline moves, it doesn't disappear.
- Ignoring state-file operational pain (locking, drift between local and remote state, state file corruption) as a reason teams evaluate Crossplane, without acknowledging Crossplane has its own operational surface (etcd load from many reconciling CRDs, provider controller resource consumption) that isn't free either.
- Assuming feature/resource parity between Crossplane providers and their Terraform provider equivalents — Crossplane provider coverage for a given cloud service can lag behind or differ from the corresponding Terraform provider, and this gap should be checked before committing to Crossplane for a specific resource type.
- Migrating everything to Crossplane at once instead of starting with the highest-value self-service use cases (frequently requested, low-risk resources) — a big-bang migration off a working Terraform estate is high risk for uncertain benefit.
