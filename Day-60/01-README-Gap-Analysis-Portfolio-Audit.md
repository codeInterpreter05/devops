# Day 60 — Phase 1 → Phase 2 Transition: Gap Analysis & Portfolio Audit

**Phase:** 1 – Core DevOps | **Week:** W9 | **Domain:** Review | **Flag:** —

## Brief

Sixty days ago you started at Linux fundamentals; today you've covered containers, Kubernetes (deep enough to sit the CKA), Terraform (deep enough to sit the Associate exam), CI/CD, cloud networking, AWS CDK, and service mesh. Phase 1 is genuinely complete — this is a review/consolidation day, not new technical content, so this note (and its companion) is intentionally short. The real work today is honest self-assessment: identifying what's actually shaky underneath a surface-level "yes I covered that," and turning your 60 days of scattered lab work into a portfolio an interviewer or hiring manager can actually look at and believe.

## Why a structured gap analysis beats "I think I know this"

The single biggest risk after a dense, fast-moving 60-day curriculum is **false confidence from recognition** — you read a topic once, it made sense at the time, and your brain files it as "known." The way this actually gets tested is production/interview conditions requiring *recall and application* under pressure, not recognition. A structured gap analysis forces the distinction:

- **Recall test**: close every note, and from memory alone, explain a topic out loud for 60 seconds (etcd's role, how NetworkPolicy default-deny works, why `terraform state mv` matters). If you can't produce a coherent 60-second explanation unaided, that's a real gap, regardless of whether the material "made sense" when you read it.
- **Application test**: given a scenario (not a definition question), can you decide what to do? "Your Deployment's pods are stuck Pending" is an application test; "what does Pending mean" is only a recall test. Interviews and real incidents are almost entirely application tests.

Go back through each week's `QUIZ.md` files across Phase 1 and specifically flag any question you couldn't answer confidently from memory, without re-reading the note first — that flagged list *is* your gap analysis, already assembled from work you already did.

## Auditing your GitHub portfolio like a hiring manager would

A hiring manager giving your GitHub 90 seconds of attention before deciding whether to read your resume more carefully is a realistic scenario — audit accordingly:

- **Does your profile README or pinned repos immediately communicate what you can do**, or does a visitor have to dig through 40 days of lab commits to figure out what you actually built? Pin the 3-5 repositories that best demonstrate range: something Kubernetes-heavy, something Terraform-heavy, something CI/CD-heavy, and ideally one that combines multiple (a Terraform-provisioned EKS cluster with an app deployed via a pipeline you wrote is a strong combined artifact).
- **Do your README files explain the "why," not just list commands you ran?** A repo that says "this Terraform config provisions a VPC" is far weaker than one that says "this provisions a VPC with public/private subnet separation and a single NAT gateway, chosen over one-per-AZ for cost, since this is a non-HA learning environment — see `README.md` for the tradeoff discussion." The second version demonstrates judgment, not just execution.
- **Is there evidence of debugging, not just successful builds?** A short write-up of a real problem you hit and how you solved it (a stuck Terraform state lock, a crash-looping pod, a broken CI pipeline) is disproportionately valuable — it's direct, undeniable evidence of the exact skill interviews test, and most portfolios have none of this.
- **Is anything embarrassing left in?** Hardcoded credentials in a committed `.tfvars` (even if the account is now deleted), a `.terraform/` directory or `node_modules/` committed by accident, commit messages like "fix" repeated 20 times with no context, or a README that's clearly copy-pasted boilerplate from a tutorial with placeholder text never replaced.

## A concrete audit checklist to run today

1. `git log --oneline` on your main learning repo — does the commit history tell a legible story, or is it noise? (You don't need to rewrite history — this is about future commits, and about whether your *pinned* repos specifically look clean.)
2. Check every pinned repo's `.gitignore` — is `.terraform/`, `*.tfstate`, `*.tfvars` (if it contains real values), `node_modules/`, and any `.env` file actually excluded? Search your git history for accidentally committed secrets (`git log -p | grep -i "AKIA\|password\|secret"` as a rough first pass, or a proper tool like `gitleaks` for a real scan).
3. Does each pinned repo have a top-level README with: what it is, why you built it, how to run/reproduce it, and one paragraph on a real tradeoff or problem you hit?
4. Cross-reference against target job postings you're interested in — do your pinned repos actually demonstrate the specific technologies those postings mention? If every posting you want mentions "Kubernetes + Terraform + CI/CD" and none of your pinned repos combine those three, that's a concrete, actionable gap to close, not a vague one.

## Points to Remember

- Recognition ("this makes sense when I read it") is not the same skill as recall ("I can explain this cold, unaided") or application ("I can decide what to do given a novel scenario") — a real gap analysis tests the latter two, not the first.
- Your own Phase 1 `QUIZ.md` files across every day are a ready-made gap-analysis tool — re-attempt them from memory and track which ones fail.
- A portfolio audit should be done from a hiring manager's realistic attention span (roughly 90 seconds per profile) — pin fewer, stronger, more-explained repos rather than exposing all 60 days of raw lab commits equally.
- Evidence of debugging a real problem (with a written account of the investigation) is more valuable in a portfolio than another clean, successful "it just worked" build.
- Scan for accidentally committed secrets/state files before anyone else finds them — this is a real, not hypothetical, risk in fast-moving learning repos.

## Common Mistakes

- Assuming "I understood it when I read it" is equivalent to being able to explain or apply it later — the two are genuinely different cognitive skills, and only the gap analysis actually distinguishes them.
- Leaving 60 days of undifferentiated commits/repos all visible with nothing pinned or curated, forcing a reviewer to guess which parts of the work actually represent your best thinking.
- Writing portfolio READMEs that describe *what* was built without ever stating *why* a particular choice was made or what tradeoff was considered — this is the single most common way a portfolio reads as "followed a tutorial" instead of "made engineering decisions."
- Never checking for committed secrets/state files in throwaway lab repos — assuming "it's just a learning repo, it doesn't matter" even though old commits with real (if now-rotated) credentials are still a bad look and a genuine security habit tell.
- Treating today as optional/skippable because it's "just review" — a poor portfolio and unaddressed gaps are exactly the kind of thing that surfaces at the worst possible time, mid-interview, if left unexamined now.
