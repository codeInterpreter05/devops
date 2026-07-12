# Day 33 — Resources: AWS IAM Deep Dive

## Primary (assigned)

- **AWS IAM documentation — "Policy evaluation logic"** (docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html) — the assigned starting point, and the authoritative source for the exact evaluation algorithm covered in file 1. Read this directly at least once; it's dense but precise.

## Deepen your understanding

- **AWS IAM documentation — "Permissions boundaries for IAM entities"** — full detail on the intersection model and the self-escalation-loophole condition-key pattern.
- **AWS Organizations documentation — "Service control policies (SCPs)"** — covers SCP inheritance through the OU tree and the "never grants, only restricts" model in depth.
- **AWS — "What Is IAM Identity Center?"** documentation — official overview of Permission Sets and account assignments.
- **AWS — "IAM Access Analyzer" documentation**, specifically the "How Access Analyzer works" page — explains the automated-reasoning approach (via the Zelkova formal-verification engine) at a level worth knowing for interviews about *how* it differs from simple rule-matching tools.

## Reference / lookup

- **AWS Policy Simulator** (policysim.aws.amazon.com) — interactive tool for testing exactly what Lab 1 and Lab 2 exercise; bookmark it, you'll use it constantly when debugging real IAM issues.
- **AWS IAM CLI reference** (`aws iam help`, `aws accessanalyzer help`) — exact syntax for every command in today's cheatsheet.

## Practice

- **AWS Skill Builder — "AWS Identity and Access Management (IAM) Fundamentals" (free digital course)** — structured, free practice reinforcing policy evaluation, boundaries, and Organizations concepts with quizzes.
