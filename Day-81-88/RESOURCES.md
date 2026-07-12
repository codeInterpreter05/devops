# Day 81-88 — Resources: Phase 2 Consolidation & Interview Prep

## Primary (assigned)

- **AWS SAA-C03 exam guide** (aws.amazon.com/certification — search "AWS Certified Solutions Architect – Associate exam guide") — the assigned starting point for this block's parallel track. It's the official domain breakdown, task statements, and weighting straight from AWS — read it before any third-party course, since every prep resource is ultimately teaching to this document.

## System design interview practice

- **"System Design Interview" by Alex Xu (Vol. 1 & 2)** — not DevOps-specific, but the requirements-first, architecture-then-tradeoffs structure it teaches is exactly the format used in [02-README-System-Design-CICD-Platform.md](02-README-System-Design-CICD-Platform.md) — useful for drilling the general muscle, not just this one question.
- **DevOps/Platform-specific mock interview practice**: DevOps-focused Discord communities and subreddits (e.g., r/devops) frequently run informal mock-interview pairing — genuinely useful because platform/infra design questions get much rarer coverage in generic system-design prep than "design Twitter"-style questions.
- **Pramp / interviewing.io** — free and paid peer mock-interview platforms; useful for getting a live, unscripted follow-up-question experience rather than just self-recording, which is a different (harder) skill than answering a static prompt.

## DevOps platform architecture case studies

- **Netflix Tech Blog** (netflixtechblog.com) — real writeups on their CI/CD (Spinnaker's origin), deployment strategies, and resilience engineering at genuinely large scale; good for grounding the "500 engineers" answer in something a real company actually built.
- **Spotify Engineering Blog** — particularly their writing on internal developer platforms and the "golden path" concept (they popularized the term) — directly informs the self-service section of the worked answer.
- **CNCF case studies** (cncf.io/case-studies) — short, concrete writeups of how various companies adopted GitOps, service mesh, and platform tooling; useful for citing a real example instead of a hypothetical when asked "have you seen this done in practice."
- **Argo CD and Flux documentation "concepts" pages** — not a case study, but both projects' docs explain the GitOps reconciliation model clearly with diagrams, and interviewers will notice if your GitOps explanation matches how it actually works versus a fuzzy approximation.

## AWS SAA-C03 study materials

- **AWS Skill Builder** (skillbuilder.aws) — AWS's own free learning platform, including an official practice question set and standard exam prep content directly aligned with the exam guide above.
- **Adrian Cantrill's SAA-C03 course** (learn.cantrill.io) — widely recommended in the DevOps/cloud community for genuinely deep, hands-on-lab-heavy AWS content rather than slide-reading; pairs well with the "hands-on beats video-watching" approach recommended in [03-README-AWS-SAA-C03-Kickoff.md](03-README-AWS-SAA-C03-Kickoff.md).
- **Tutorials Dojo SAA-C03 practice exams** — a well-regarded practice-question bank specifically for calibrating readiness (used for the baseline practice exam in Days 7-8 of the lab); known for practice questions that closely mirror the real exam's scenario-based style, plus detailed answer explanations.
- **AWS Free Tier** (aws.amazon.com/free) — required for the hands-on portion of the kickoff (VPC, EC2 + ALB + Auto Scaling group, S3 lifecycle policy); set a billing alarm before you start experimenting.

## Reference / lookup

- **AWS Well-Architected Framework** (docs.aws.amazon.com/wellarchitected) — the five/six pillars underpin a large share of SAA-C03's "which architecture is best" scenario questions; worth skimming even before deep-diving individual services.
- **explainshell.com / man pages** — same on-box reference habit from Day 1, still useful when a hands-on AWS CLI lab throws an unfamiliar flag.
