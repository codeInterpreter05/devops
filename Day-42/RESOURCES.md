# Day 42 — Resources: K8s + IaC + AWS Mock Interview

## Primary (assigned)

- **Grokking System Design (free summaries on GitHub)** — the assigned starting point. While not Kubernetes/AWS-specific, its structured approach to requirement-clarification and component trade-off discussion directly maps onto the framework used in today's system-design notes.

## Deepen your understanding

- **AWS Well-Architected Framework — Reliability & Performance Efficiency pillars** (aws.amazon.com/architecture/well-architected) — the closest thing to an "official" checklist for the trade-offs this week's system-design file leans on (Multi-AZ, auto-scaling, caching, connection pooling).
- **Terraform official docs — State** (developer.hashicorp.com/terraform/language/state) — the authoritative reference for `state mv`/`rm`/`import`, refresh behavior, and locking, used throughout the state-debugging review.
- **Kubernetes official docs — Using RBAC Authorization** (kubernetes.io/docs/reference/access-authn-authz/rbac) — the canonical reference for subresources, aggregated ClusterRoles, and the Role/ClusterRole/Binding scoping rules reviewed today.
- **"Designing Data-Intensive Applications" (Martin Kleppmann)** — not free, but the single best deep resource for the CAP-theorem/consistency trade-offs that come up in any "globally available" system design answer; even reading the replication/partitioning chapters pays off directly here.

## Reference / lookup

- **Karpenter documentation** (karpenter.sh) — node-level autoscaling specifics referenced in the EKS system-design file.
- **Argo Rollouts documentation** (argoproj.github.io/argo-rollouts) — canary/blue-green deployment patterns referenced in the zero-downtime-deploys discussion.
- **`kubectl auth can-i` reference** (kubernetes.io CLI docs) — full flag list for RBAC troubleshooting.

## Practice

- **Complete today's full mock-interview lab** (45-minute timed system design + live Terraform state debugging + K8s RBAC exercise) with a study partner acting as interviewer — this is the assigned hands-on activity, and the time pressure is the point; reviewing the material without the timed component misses what's actually being tested.
- **Pramp / interviewing.io** (free peer mock-interview platforms) — schedule a real system-design mock with a stranger to get comfortable narrating trade-offs out loud under genuine unfamiliar-interviewer pressure, not just with a study partner who already knows the "answer."
