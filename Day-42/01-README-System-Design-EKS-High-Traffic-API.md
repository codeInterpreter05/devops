# Day 42 — Review: System Design — High-Traffic API on EKS

**Phase:** 1 – Core DevOps | **Week:** W6 | **Domain:** Review | **Flag:** 📌 Review/consolidation day

## Brief

This is the first of three review files closing out Phase 1's Core DevOps block — no new tool is introduced today. Instead, this file rehearses the single most common senior-DevOps/SRE interview format: a timed system-design exercise, specifically *"Design the complete AWS architecture for a globally available microservices app"* / *"Design AWS infra for a 10k RPS API with auto-scaling, HA, and zero-downtime deploys."* The goal isn't to memorize one "correct" diagram — it's to build a repeatable **framework** for structuring an answer under time pressure, pulling together everything from Weeks 1–6 (Linux, containers, Terraform, Ansible, Kubernetes, secrets, networking) into one coherent design.

## A framework for answering system design questions

Interviewers grade the *process* as much as the final architecture. A strong 45-minute answer moves through these stages, narrating trade-offs out loud the whole way, rather than jumping straight to a final diagram:

1. **Clarify requirements first** (2–3 minutes) — RPS (10k requests/sec — read-heavy? write-heavy? mixed?), latency SLA, global vs. single-region, data consistency needs, budget/team-size constraints. A candidate who starts drawing boxes before asking "is this read-heavy or write-heavy" is optimizing the wrong things later.
2. **High-level architecture** (10 minutes) — the major building blocks and how traffic flows through them, before drilling into any one piece.
3. **Deep-dive on 2–3 components** the interviewer probes hardest — usually compute/scaling, data layer, and deployment strategy.
4. **Explicitly call out trade-offs and failure modes** — nothing in system design is free; naming the cost of each choice (money, complexity, latency) is what separates a strong answer from a shopping list of AWS services.

## High-level architecture for a 10k RPS API on EKS

```
Route 53 (latency-based / geo routing, health checks)
   -> CloudFront (CDN, edge caching, static assets, DDoS absorption via AWS Shield)
      -> ALB (Application Load Balancer, per region)
         -> EKS cluster (multiple AZs, managed node groups + Karpenter/Cluster Autoscaler)
            -> Ingress-NGINX / AWS Load Balancer Controller -> Services -> Pods (HPA-scaled)
               -> RDS Aurora (Multi-AZ, read replicas) / DynamoDB (for high-throughput, simple-access-pattern data)
               -> ElastiCache (Redis) for caching/session state
               -> S3 for object storage/static assets
Secrets: AWS Secrets Manager / Vault, injected via IRSA + ESO
Observability: Prometheus + Grafana, CloudWatch, centralized logging (Loki/ELK), distributed tracing (X-Ray/Jaeger)
CI/CD: GitHub Actions/GitLab CI -> ECR -> ArgoCD/Flux (GitOps) -> EKS
```

### Compute and auto-scaling — the core of the "10k RPS + auto-scaling" requirement

- **EKS with managed node groups across ≥3 AZs** for baseline capacity, plus **Karpenter** (or Cluster Autoscaler) for fast, workload-aware node provisioning — Karpenter is the modern default recommendation because it provisions right-sized nodes directly matching pending pod requirements, faster than Cluster Autoscaler's group-based scaling.
- **Horizontal Pod Autoscaler (HPA)** scaling on CPU/memory or, better, **custom metrics** (requests-per-second via Prometheus Adapter) — CPU-based scaling alone is a common weak point to call out: a request-heavy but CPU-light service won't trigger HPA fast enough on CPU alone.
- **Pod Disruption Budgets (PDBs)** to ensure node scale-down/upgrades don't drop below a minimum healthy replica count during voluntary disruptions.
- Multi-AZ by default — losing one AZ should not take the service down, which requires the ALB, EKS node groups, and the data layer (Aurora Multi-AZ) to all independently tolerate an AZ failure.

### Data layer trade-offs — the part interviewers probe hardest

- **Aurora (Postgres/MySQL-compatible)**: strong consistency, relational modeling, Multi-AZ failover in under a minute, read replicas for read-heavy scaling — the default choice unless access patterns are simple key-value at very high throughput.
- **DynamoDB**: near-infinite horizontal scaling, single-digit millisecond latency, but requires access patterns designed up front (no ad-hoc joins/queries) — a strong answer flags that switching to DynamoDB *after* the fact, once query patterns are already relational, is expensive to retrofit.
- **ElastiCache (Redis)** in front of either, for hot-path caching — explicitly reduces database load and latency for repeated reads, at the cost of cache-invalidation complexity (a real, nameable trade-off, not a free win).
- **Read replicas + connection pooling (RDS Proxy/PgBouncer)** — at 10k RPS, an unpooled connection-per-request pattern will exhaust database connection limits long before compute is the bottleneck; this is a common trap candidates miss if they only reason about compute scaling.

### Zero-downtime deploys — directly answering the "HA and zero-downtime deploys" part of the prompt

- **Rolling updates** (Kubernetes' default Deployment strategy) with a correctly tuned `readinessProbe` — a pod is only added to Service endpoints once it reports ready, and `maxUnavailable`/`maxSurge` control how aggressively old pods are replaced.
- **Blue/green or canary** (via Argo Rollouts or a service mesh like Istio/Linkerd) for higher-risk changes — shifting a small percentage of traffic to the new version first, with automated rollback on error-rate/latency regression, is the strong answer for "how do you deploy with zero downtime *and* minimize blast radius," versus a plain rolling update which only addresses the "zero downtime" half.
- **Database migrations** handled as expand/contract (add new columns/tables additively, deploy code that can read both old and new shapes, backfill, then remove old shape in a later release) — a frequently-missed point: zero-downtime deploys are only as safe as the riskiest schema migration riding along with them.

## Points to Remember

- Always clarify requirements (RPS shape, latency SLA, consistency needs, single vs. multi-region) before drawing any architecture — this is graded as heavily as the final design.
- Karpenter/Cluster Autoscaler handles node-level scaling; HPA handles pod-level scaling — a complete auto-scaling answer addresses both layers, not just one.
- Naming a concrete trade-off for every major choice (Aurora vs. DynamoDB, caching vs. invalidation complexity, canary vs. plain rolling update) is what separates a senior-level answer from a service-name checklist.
- Zero-downtime deploys require correct readiness probes, sane rolling-update parameters, and — for anything with a schema change — an expand/contract migration strategy, not just a deployment strategy alone.
- Connection pooling and read replicas are frequently the actual bottleneck at high RPS, not raw compute — naming this proactively is a strong signal of real production experience.

## Common Mistakes

- Jumping straight into drawing boxes without clarifying requirements first — designing for global multi-region HA when the actual requirement was a single-region 10k RPS API wastes time and signals a template-following approach rather than genuine reasoning.
- Treating "auto-scaling" as a single concept — forgetting that Kubernetes' pod-level HPA and the cluster's node-level autoscaler are two separate, both-necessary mechanisms.
- Choosing DynamoDB by default "because it scales" without acknowledging its access-pattern rigidity, or choosing Aurora by default without acknowledging its comparatively harder horizontal write-scaling ceiling.
- Describing "zero-downtime deploys" as solved purely by Kubernetes rolling updates, while ignoring that an accompanying breaking database migration can still cause an outage regardless of how gracefully the application pods themselves roll over.
- Not mentioning connection pooling/read replicas at all when discussing a 10k RPS data layer — a very common gap that becomes the first thing an experienced interviewer probes on.
