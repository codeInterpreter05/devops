# Day 59 — Phase 1 Mock Interview Day: Format & System Design Practice

**Phase:** 1 – Core DevOps | **Week:** W9 | **Domain:** Review | **Flag:** 📌

## Brief

You've spent 8 weeks building real technical depth — Linux, containers, Kubernetes, Terraform, CI/CD, cloud networking, service mesh, IaC on AWS. None of that converts into a job offer if you can't structure it into a clear, confident answer under interview conditions in real time. Today is a deliberate stop to practice the *delivery* skill, not new technical content: running a real 60-minute mock interview, and specifically drilling the system-design format using this phase's capstone-style prompt — designing a multi-tenant SaaS platform on EKS with tenant isolation. This is a review/consolidation day, so this file (and the companion scenario file) is your only reading — the real work today is doing the mock interview, recording it, and critiquing yourself honestly.

## Why a structured mock interview format matters

DevOps/platform engineering interviews at any company with a mature hiring bar tend to follow a predictable shape, and knowing the shape in advance is a real, legitimate advantage — not "gaming" the interview, since the structure exists precisely because it's how experienced engineers actually reason about unfamiliar problems live:

1. **Behavioral/background** (5-10 min) — walk through your experience, usually anchored on a specific project ("tell me about a time you debugged a production incident").
2. **System design** (20-30 min) — an open-ended architecture prompt, evaluated on how you *structure ambiguity*, not on producing one "correct" diagram.
3. **Hands-on/scenario debugging** (15-20 min) — "here's a broken cluster/pipeline/Terraform state, walk me through how you'd investigate," testing real operational instinct rather than memorized commands.
4. **Your questions for them** (5 min) — a real signal too; asking about on-call load, deployment frequency, or incident postmortem culture reads as someone who's actually operated systems.

A full 60-minute mock covering all four in one sitting — timed, recorded, self-reviewed — is dramatically more useful than answering the same questions untimed with your notes open, because interview performance under time pressure and without the safety net of "let me look that up" is a distinct skill from knowing the material.

## System design interviews: the framework, applied

The trap in system design interviews is diving straight into a specific technology ("we'll use EKS with Istio and...") before establishing what you're actually optimizing for. A strong structure, in order:

1. **Clarify requirements first, out loud.** For "design a multi-tenant SaaS platform on EKS with tenant isolation": How many tenants, roughly (10s? 1000s?)? Is isolation a compliance requirement (some tenants need hard guarantees, e.g. regulated industries) or just a "don't let one noisy tenant affect another" concern? Is this B2B (few large tenants) or B2C-at-scale (many small tenants)? These answers completely change the right architecture, and asking them signals real experience — nobody senior designs a multi-tenant system without asking these questions first.

2. **State your isolation model choice, and justify the tradeoff.** For EKS multi-tenancy, the real options, cheapest-to-most-isolated:
   - **Namespace-per-tenant** (same cluster, same nodes): cheapest, fastest to provision, isolation enforced via RBAC + NetworkPolicy + ResourceQuotas — but a kernel-level or node-level compromise/noisy-neighbor issue can still cross tenant boundaries since containers share the host kernel.
   - **Node-pool-per-tenant (or per-tenant-tier)** with taints/tolerations and node affinity: stronger blast-radius isolation for "noisy neighbor" resource contention, still shares the control plane and network fabric.
   - **Cluster-per-tenant**: strongest isolation (separate control plane, fully separate blast radius for a cluster-level incident), but the most expensive and highest operational overhead — reasonable only for a handful of large, compliance-sensitive tenants, not thousands of small ones.
   - Mention that most real production multi-tenant SaaS platforms use a **hybrid**: namespace-per-tenant for the majority, with an escape hatch to dedicated node pools or dedicated clusters for large/regulated tenants who pay for or require it.

3. **Cover the concrete isolation mechanisms**, not just the model name: `ResourceQuota`/`LimitRange` per namespace (CPU/memory/object-count caps so one tenant can't exhaust the cluster), `NetworkPolicy` default-deny between tenant namespaces, per-tenant IAM roles via IRSA (IAM Roles for Service Accounts) so a tenant's workload can only reach its own S3 bucket/DynamoDB table and not another tenant's, and Pod Security Standards/admission control to stop privilege escalation.

4. **Address data layer isolation separately from compute** — this is the part candidates most often forget. Shared RDS instance with per-tenant schemas/row-level security? Separate database per tenant? DynamoDB with a tenant-ID partition key and IAM condition keys scoping access? This decision is at least as important as the Kubernetes-level isolation and interviewers specifically probe whether you remember data isolation isn't automatically solved by namespace isolation.

5. **Talk about the control plane you'd add on top**: an ingress/API gateway that resolves tenant identity (subdomain, header, JWT claim) and routes accordingly; a provisioning/control-plane service that creates a new tenant's namespace + quotas + RBAC + IAM role automatically (never manually) when a new customer signs up; observability that's tenant-aware (can you answer "which tenant is causing this latency spike" quickly).

6. **Close with tradeoffs and what you'd measure** — cost per tenant, blast radius of a cluster-level incident, and what would make you reconsider the isolation tier for a specific tenant (e.g., a customer's compliance requirement forcing a move from shared namespace to dedicated cluster).

**What interviewers are actually scoring:** not whether you picked "the right" isolation model (there often isn't one correct answer), but whether you asked clarifying questions before committing, can articulate the tradeoff of your choice honestly, and remembered that "isolation" spans compute, network, data, and identity — not just one Kubernetes primitive.

## Points to Remember

- The four-part interview shape (behavioral, system design, hands-on debugging, your questions) is common enough across the industry that rehearsing it explicitly is a legitimate, not "cheating," preparation strategy.
- In system design, clarify scale/compliance/tenant-shape requirements *before* naming technologies — jumping straight to "we'll use Istio" without establishing what problem you're solving is the most common junior tell.
- Multi-tenant isolation on Kubernetes is a spectrum (namespace → node-pool → cluster-per-tenant), not a binary choice, and most real systems are hybrids based on tenant tier.
- Data-layer isolation (database/schema/row-level security, tenant-scoped IAM) is a distinct decision from compute isolation and is the piece candidates most often forget to mention.
- Recording yourself and reviewing the playback is uncomfortable but disproportionately effective — you will hear filler words, unclear structure, and rambling that you don't notice while speaking live.

## Common Mistakes

- Launching straight into a specific tool/architecture before asking a single clarifying question about scale, tenant shape, or compliance needs.
- Treating "multi-tenant isolation" as solved purely by `kubectl create namespace` — forgetting NetworkPolicy, ResourceQuota, and IAM/data isolation entirely.
- Presenting only one architecture with no discussion of tradeoffs or alternatives — system design answers that sound like a memorized diagram rather than live reasoning are an easy negative signal.
- Never mentioning cost or operational overhead — a cluster-per-tenant answer with no acknowledgment of its expense/overhead reads as inexperienced with running real infrastructure.
- Skipping the "your questions for them" portion of a mock, or asking only shallow questions ("what's the tech stack") instead of ones that reveal how the team actually operates (on-call rotation, deployment frequency, incident review process).
