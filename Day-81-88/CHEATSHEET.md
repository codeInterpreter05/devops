# Day 81-88 — Cheatsheet: Phase 2 Consolidation & Interview Prep

## System-design-answer framework (use this checklist for any design question, not just the CI/CD one)

```
1. REQUIREMENTS  (spend 5-10 min here — don't skip this)
   - Functional: what must the system actually do?
   - Non-functional: scale (users/engineers/requests), latency,
     availability, compliance/regulatory constraints
   - Ask clarifying questions; state assumptions explicitly if unanswered

2. HIGH-LEVEL ARCHITECTURE
   - Draw the end-to-end flow, box by box
   - Name the major components, not implementation-level detail yet
   - Identify the data flow direction (push vs. pull, sync vs. async)

3. DEEP DIVE (interviewer usually steers you here)
   - Pick 1-2 components to go deep on
   - Self-service / scaling story if org-scale is part of the question
   - Failure modes: what happens when this component is slow/down?

4. SECURITY
   - Where are the gates? (as early/cheap as possible, ordered by cost)
   - AuthN/AuthZ, secrets handling, audit trail
   - Defense in depth — don't rely on a single layer

5. TRADEOFFS  (say these UNPROMPTED — this is what separates senior
   answers from junior ones)
   - Buy vs. build
   - Centralized vs. federated ownership
   - Consistency vs. availability, speed vs. safety
   - What would change at 10x the stated scale?
```

## Mock-interview self-grading rubric

Score each dimension 1-5 (1 = didn't do this, 5 = did this smoothly and unprompted). A strong session averages 4+; anything scoring 2 or below is a flagged weakness to target in your next session.

| Dimension | 1 (weak) | 3 (okay) | 5 (strong) |
|---|---|---|---|
| **Requirements gathering** | Jumped straight to solution | Asked a couple of clarifying questions | Systematically covered functional + non-functional before designing anything |
| **Structure/communication** | Rambled, jumped around, hard to follow | Mostly linear, some backtracking | Clear signposting ("first I'll cover X, then Y"), easy to follow without notes |
| **Depth on deep-dive** | Vague, hand-wavy on the one area probed | Correct but surface-level | Concrete detail, named real tools/mechanisms, explained *why*, not just *what* |
| **Security awareness** | Not mentioned unless directly asked | Mentioned generically ("we'd add security") | Specific gates named, ordered logically, tied to real threats |
| **Tradeoffs stated** | None offered | Offered only when asked | At least 2-3 stated unprompted, with reasoning |
| **Time management** | Ran out of time before covering key areas / finished in 10 min with huge gaps | Roughly on pace | Used the full time well, left room for interviewer follow-ups |
| **Behavioral/STAR answers** | No clear structure, missed the "result"/impact | Structure present but thin on result/impact | Clear Situation-Task-Action-Result, quantified impact where possible |
| **Composure under follow-up** | Froze or got defensive on a hard follow-up | Recovered but visibly rattled | Treated follow-ups as normal, adjusted the answer smoothly |

**Overall score = average of the 8 dimensions.** Track this number across session 1 and session 2 — the goal for this block is a measurable increase, not a perfect score.

## AWS SAA-C03 domain breakdown

| # | Domain | Weight | Core topics |
|---|---|---|---|
| 1 | Design Resilient Architectures | ~30% | Multi-AZ/region, decoupling (SQS/SNS), Auto Scaling, RDS Multi-AZ vs. read replicas, DR strategies (backup/restore, pilot light, warm standby, multi-site) |
| 2 | Design High-Performing Architectures | ~28% | S3 storage classes, EBS types, CloudFront/ElastiCache caching, RDS vs. DynamoDB vs. Aurora, enhanced networking/placement groups |
| 3 | Design Secure Applications and Architectures | ~24% | IAM (least privilege, roles/policies), security groups vs. NACLs, KMS/encryption in transit and at rest, VPC segmentation |
| 4 | Design Cost-Optimized Architectures | ~18% | Lifecycle policies, Reserved/Spot/Savings Plans vs. On-Demand, right-sizing, cheapest-service-that-meets-requirement thinking |

**Exam facts:** 65 questions (multiple-choice + multiple-response) · 130 minutes · scaled score 100-1000 · pass at 720 · no guessing penalty — always answer.

## Deployment strategy quick comparison

```
ROLLING     : replace instances/pods gradually, old + new coexist briefly
              optimizes for: no extra infra cost
              rollback: slower (has to roll back the same way)

BLUE-GREEN  : two full environments, instant traffic cutover
              optimizes for: fastest rollback (flip traffic back)
              cost: needs double infrastructure during cutover

CANARY      : small % of traffic to new version, expand gradually
              optimizes for: earliest real-traffic signal, smallest blast radius
              needs: traffic splitting + automated metric-based promotion/abort
```

## DevSecOps scan-type quick reference

```
SAST  — Static Application Security Testing   — scans source code, pre-build
SCA   — Software Composition Analysis          — scans dependencies for known CVEs
DAST  — Dynamic Application Security Testing   — scans a running app (black-box)
IAST  — Interactive AST                        — instrumented, runs during tests
Secret scanning — catches leaked credentials in commits/history
Image scanning  — catches vulnerable OS/library layers in a built container image
```

## Supply chain security quick reference

```
SBOM      — Software Bill of Materials: a manifest of everything in an artifact
Signing   — cosign/Sigstore: cryptographically verify an artifact wasn't tampered with
SLSA      — a framework/ladder of provenance levels (how/where/from-what an artifact was built)
Pin by digest, not tag — tags are mutable; a digest (sha256:...) is immutable
```
