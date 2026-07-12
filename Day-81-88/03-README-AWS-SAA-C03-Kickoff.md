# Day 81-88 — Phase 2 Consolidation III: Starting AWS SAA-C03 Prep in Parallel

**Phase:** 2 – CI/CD & Security | **Week:** W13-W14 | **Domain:** Review & Buffer | **Flag:** 📌

## Brief

Phase 3 assumes solid AWS fundamentals, and the AWS Certified Solutions Architect – Associate (SAA-C03) is the assigned "best resource" for this block — not as a side quest, but as the structured syllabus that tells you exactly what to know before you're deep into cloud-native/AWS-heavy days. The trick during this specific 8-day block is **timeboxing**: you're also running mock interviews, writing a system design doc, and fixing portfolio gaps (see [LAB.md](LAB.md)). SAA-C03 prep needs to start now, in parallel, without eating the hours those higher-priority activities need this week. This file gives you the exam's actual shape and a realistic way to carve out study time without it becoming the thing that quietly consumes the whole block.

## Exam structure — know this before you study a single domain

- **65 questions**, mix of multiple-choice (one correct answer) and multiple-response (select N correct answers).
- **130 minutes.**
- Scored on a scaled range of **100-1000**; passing score is **720**.
- **No penalty for guessing** — never leave a question blank, always select something before moving on.
- Delivered via Pearson VUE, in-person or remote-proctored.
- Recertifies every 3 years.

Knowing this shape matters practically: at 65 questions in 130 minutes, you have exactly **2 minutes per question** — the exam is testing applied judgment on realistic scenarios, not recall of trivia, so during practice tests train yourself to eliminate obviously-wrong options fast rather than re-reading every option exhaustively.

## The four domains and their weight

| Domain | Weight | What it actually covers |
|---|---|---|
| **1. Design Resilient Architectures** | ~30% | Multi-AZ/multi-region resilience, decoupling (SQS/SNS), auto scaling, RDS Multi-AZ vs. read replicas, backup/DR strategies (backup & restore, pilot light, warm standby, multi-site). |
| **2. Design High-Performing Architectures** | ~28% | Storage tiering (S3 storage classes, EBS types), caching (CloudFront, ElastiCache), database choice (RDS vs. DynamoDB vs. Aurora), networking performance (placement groups, enhanced networking). |
| **3. Design Secure Applications and Architectures** | ~24% | IAM (users/roles/policies, least privilege), VPC security (security groups vs. NACLs, subnets), KMS/encryption at rest and in transit, securing data at multiple layers. |
| **4. Design Cost-Optimized Architectures** | ~18% | Storage lifecycle policies, Reserved/Spot/Savings Plans vs. On-Demand, right-sizing, choosing the cheapest service that meets the requirement. |

Domain 4 is the one most self-studiers under-invest in because it "feels boring" next to resilience/security — don't skip it; at ~18% it's roughly one in five questions.

## Where your existing DevOps knowledge already overlaps

A meaningful chunk of SAA-C03 content is stuff you've already touched hands-on in Phases 0-2, just not always through an AWS-specific lens:

- **IAM** — you've used roles/policies for CI/CD pipeline permissions and least-privilege service accounts; SAA-C03 formalizes this into policy JSON structure, role assumption, and permission boundaries.
- **VPC networking basics** — security groups (stateful, instance-level) vs. NACLs (stateless, subnet-level) is a near-guaranteed exam topic, and if you've configured network policies for Kubernetes or firewalled a CI runner, the mental model transfers directly.
- **EC2/Auto Scaling/ELB** — if you've deployed anything behind a load balancer with health checks, you already understand the shape of Domain 1's scaling questions; SAA-C03 adds the AWS-specific vocabulary (target groups, launch templates, scaling policies).
- **S3** — if you've used S3 as an artifact store or Terraform state backend, storage classes and lifecycle policies (Domain 2 and 4) will feel like labeling something you already use.

Don't skip these topics assuming "I already know this" — SAA-C03 questions test the AWS-specific edge cases (e.g., exactly which storage class transitions are allowed, exactly what a NACL evaluates that a security group doesn't) more than the general concept.

## How to timebox this in parallel with mock interviews + portfolio work

This block's primary deliverables (2 mock interviews, the system design doc, portfolio fixes) take priority — SAA-C03 prep should fit around them, not compete head-on. A realistic split across the 8 days:

- **A fixed 45-60 minute daily slot**, same time each day if possible (e.g., first thing before the day's interview-prep work) — protects it from being crowded out without letting it expand to eat the whole day.
- **Days 1-3 of this block**: skim the official exam guide's domain breakdown (see RESOURCES.md) and watch a domain-1/domain-2 overview — get the map before the details.
- **Days 4-6**: hands-on in the AWS free tier — stand up a VPC with public/private subnets, launch an EC2 instance behind an ALB with an Auto Scaling group, create an S3 bucket with a lifecycle policy. Doing beats watching for retention.
- **Days 7-8**: take one full-length practice exam cold (don't study the answers yet — this is a baseline, not a study method) to see where you actually stand before Phase 3 begins, and use the wrong answers to build your real study list for the weeks ahead.
- Treat this block's SAA-C03 work as **kickoff, not completion** — the real study plan continues into Phase 3; the goal here is momentum and a baseline, not mastery.

## Points to Remember

- 65 questions, 130 minutes, scaled score out of 1000, passing at 720, no guessing penalty — always answer every question.
- Domain weights: Resilient (~30%) > High-Performing (~28%) > Secure (~24%) > Cost-Optimized (~18%) — don't neglect the smallest-weighted domain, it's still ~1 in 5 questions.
- A lot of SAA-C03 content overlaps with hands-on DevOps work you've already done — but the exam tests AWS-specific mechanics, not just the general concept, so don't skip "familiar" topics.
- Protect a fixed daily time slot for this rather than letting it expand or contract based on how the rest of the day goes — consistency over intensity during a busy consolidation week.
- This block's goal is a baseline practice-exam score and hands-on familiarity, not full exam readiness — that continues in parallel with Phase 3.

## Common Mistakes

- Starting with a full video course end-to-end before any hands-on practice or practice questions — passive video consumption for AWS certs has a notably weak correlation with actually passing, compared to hands-on labs + practice exams.
- Letting SAA-C03 prep expand past its timeboxed slot and eat into mock-interview or portfolio-fix time during this specific block — this block has explicit higher-priority deliverables.
- Skipping the cost-optimization domain because it feels less technically interesting than resilience/security — it's worth roughly the same as the security domain.
- Assuming DevOps experience fully covers the exam — general cloud/container knowledge transfers conceptually, but specific AWS service limits, exact IAM policy evaluation logic, and storage class transition rules need dedicated study.
- Taking a practice exam and immediately reviewing every answer before attempting a second cold attempt later — this destroys the value of a practice exam as a baseline measurement.
