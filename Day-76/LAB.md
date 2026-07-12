# Day 76 — Lab: CI/CD & DevSecOps Review

**Goal:** Turn Phase 2 review into concrete artifacts — a documented gap list with closed gaps, and 3 real security-improvement PRs filed against a public GitHub Actions pipeline.

**Prerequisites:**
- Your own notes/repo for Days 61–75 (this repo).
- A GitHub account with a fork workflow set up (`git`, `gh` CLI authenticated).
- 60–90 minutes of uninterrupted time — this lab is deliberately slower and more reflective than a tool-installation lab.

---

### Lab 1 — Closed-book topic dump

1. Set a 5-minute timer. For each of these Phase 2 domains, write from memory (no notes) 3-5 bullet points of what you actually remember: CI pipeline gating design, SAST/SCA, container image scanning, secrets management in CI, artifact signing/provenance, GitOps (ArgoCD), compliance-as-code (CIS/Prowler/SOC2/GDPR), database migrations (expand-contract).
2. Now open your actual notes for each domain and compare. Circle anything you wrote incorrectly or couldn't recall at all.
3. Create a running `gaps.md` file listing every circled item.

**Success criteria:** A `gaps.md` file with at least one concrete, specific gap per domain (not "I forgot everything about X" — the specific fact/mechanism you got wrong or blanked on).

---

### Lab 2 — Explain-out-loud + interview question cross-check

1. Pick 3 items from your gap list. Explain each out loud, unscripted, for 90 seconds, as if to an interviewer. Record audio if you can.
2. Listen back (or just reflect immediately) and note every hedge phrase ("I think," "something like," "I'm not 100% sure but"). Those are your real priority gaps.
3. Go through the "Interview Question to prep" line from each of Days 61–75's spreadsheet rows (or the QUIZ.md interview question in each folder) and answer 5 of them cold, under 2 minutes each, timed.

**Success criteria:** You've identified, by ear, at least 3 specific topics where your explanation was noticeably hedgy or vague — not just topics you "felt fine" about.

---

### Lab 3 — Close the top 3 gaps with a concrete example

1. For your top 3 highest-priority gaps (from Labs 1–2), do one of: re-run a relevant lab command, write a fresh example config/snippet from scratch without copying, or write 3-4 sentences explaining the mechanism in your own words with a concrete example attached.
2. Update the relevant Day's notes (or your `gaps.md`) with that concrete example and a link back to the source day.

**Success criteria:** Your notes for the 3 closed gaps now contain a real, self-authored example — not just a restated definition.

---

### Lab 4 — Audit a real open-source pipeline

1. Pick a mid-sized, actively maintained open-source repo with a non-trivial `.github/workflows/` directory (a project you use, or search GitHub for popular repos with CI).
2. Run through the audit checklist in `02-README-Pipeline-Audit-Optimisation.md` — supply-chain, secrets handling, gating, efficiency.
3. Identify at least 5 concrete findings (more than you'll actually PR — pick the 3 best afterward).

**Success criteria:** A written list of at least 5 specific, cited findings (file + line reference) against the checklist.

---

### Lab 5 — File 3 real security-improvement PRs

1. Fork the repo. For each of your 3 best findings, make a small, focused change (e.g., pin one Action to SHA, fix one `exit-code: 0` scan, add one missing `needs:` gate).
2. Write a clear PR description: what's the risk, why this specific fix addresses it, and any trade-off it introduces (e.g., "this pins to SHA — you'll need Dependabot or manual bumps going forward").
3. Open all 3 PRs. (It's fine if a maintainer never responds or declines — the goal is the audit-and-fix skill and the artifact, not a guaranteed merge.)

**Success criteria:** 3 open PRs, each addressing one specific, correctly-diagnosed issue with a clear, professional description a maintainer could act on without asking clarifying questions.

---

### Cleanup

```bash
# No cloud resources to tear down today — this is a documentation/review day.
# Keep gaps.md and the 3 PR links as permanent artifacts, don't delete them.
```

### Stretch challenge

Turn your audit checklist into a reusable script: a small shell/Python tool that greps a `.github/workflows/*.yml` file for mutable-tag `uses:` lines, `pull_request_target` combined with fork checkout, and `exit-code: '0'`/`continue-on-error: true` on scan steps, printing a findings report automatically — then run it against 3 more repos to see how common these issues really are in the wild.
