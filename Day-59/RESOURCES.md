# Day 59 — Resources: Phase 1 Mock Interview Day

## Primary (assigned)

- **Your own notes** — the assigned resource for today. Re-read your Day 1-58 notes (especially the `QUIZ.md` interview questions and `Common Mistakes` sections across every day) as your actual study material; today is about applying eight weeks of accumulated notes under interview conditions, not consuming new content.

## Deepen your understanding

- **"Designing Data-Intensive Applications" by Martin Kleppmann** (O'Reilly) — not DevOps-specific, but the best available grounding in the tradeoffs (consistency, partitioning, replication) that come up constantly in system design interviews, including the data-isolation half of today's multi-tenant question.
- **AWS Well-Architected Framework — SaaS Lens** (aws.amazon.com/architecture/well-architected) — AWS's own published guidance specifically on multi-tenant SaaS design patterns, tenant isolation tiers, and cost-per-tenant modeling — directly maps onto today's system design prompt.
- **"System Design Interview" by Alex Xu (Volumes 1 & 2)** — widely used prep material for structuring open-ended design answers; the general framework (clarify → high-level design → deep dive → tradeoffs) transfers directly to infrastructure-flavored prompts like today's.
- **Google SRE Book — "Effective Troubleshooting" chapter** (sre.google/sre-book) — the best free treatment of hypothesis-driven debugging discipline, the exact skill Scenario 1 and 2 in today's notes are drilling.

## Reference / lookup

- **Kubernetes docs — Multi-tenancy** (kubernetes.io/docs/concepts/security/multi-tenancy) — the official reference enumerating isolation mechanisms (namespaces, RBAC, NetworkPolicy, ResourceQuota, node isolation) referenced in today's system design answer.
- **Terraform docs — CLI: force-unlock** (developer.hashicorp.com/terraform/cli/commands/force-unlock) — read this once so the exact semantics and risk of the command are fresh before you ever need to reach for it under pressure.

## Practice

- **Pramp / interviewing.io** — free or low-cost peer mock interview platforms where a stranger plays interviewer and can ask real, unscripted follow-up questions — meaningfully harder and more useful than a fully solo run.
- **Record a second mock interview in 1-2 weeks** and compare it against today's recording side by side — the delta between the two is the actual signal of whether your practice is working, not either recording in isolation.
