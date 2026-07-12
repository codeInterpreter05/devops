# Day 54 — CKA Exam Prep I: Exam Format & Strategy

**Phase:** 1 – Core DevOps | **Week:** W8 | **Domain:** Certifications | **Flag:** 📌 Milestone

## Brief

The Certified Kubernetes Administrator (CKA) exam is unlike almost every other IT certification you've probably taken — it's not multiple choice, it's a **live, timed, hands-on** exam where you solve real problems in a real terminal against real clusters. That format is exactly why "know Kubernetes" isn't sufficient prep — you need **speed and exam-specific tactics** layered on top of genuine understanding. Today is about internalizing the format and rules so nothing about exam day itself is a surprise, freeing all your mental bandwidth on exam day for the actual technical tasks.

This day is split into three files:

1. **This file** — exam format, timing, allowed resources, and overall strategy.
2. **[02-README-kubectl-Speed-Tricks.md](02-README-kubectl-Speed-Tricks.md)** — the fastest imperative `kubectl` commands and aliases that actually move the needle under time pressure.
3. **[03-README-Practice-Environment.md](03-README-Practice-Environment.md)** — setting up practice clusters with `kind` and using killer.sh effectively.

## Exam format — the concrete numbers

- **17 performance-based tasks** (the exact count and exact task list vary by exam version, but this is the current ballpark) — each task is an independent problem, usually scoped to a specific cluster/context you `kubectl config use-context` into at the start of that task.
- **2 hours** total — that's roughly **7 minutes per task on average**, though tasks are explicitly **not weighted equally**; some are worth 2-3x more than others. This is the single most important strategic fact about the exam: **spend your time proportional to a task's point value, not proportional to how "interesting" or familiar it feels.**
- **Passing score: 66%** (current threshold; historically has moved slightly between exam versions — always confirm the current number on the CNCF/Linux Foundation exam page before your actual sitting).
- Delivered via the **PSI Secure Browser**, remotely proctored — you're webcam-monitored the entire time, in a room that must meet specific rules (clear desk, no other monitors, no phone within reach, etc. — check the current candidate handbook, since proctoring rules are strict and candidates have been flagged for violations they didn't realize were rule-breaking).
- Exam environment gives you **multiple separate clusters/contexts** across the 17 tasks — part of the skill being tested is correctly switching context at the start of each task and confirming you're operating against the right cluster before touching anything.

## Allowed documentation during the exam

You are permitted **one browser tab** with access to a fixed, specific set of Kubernetes documentation domains:
- `kubernetes.io/docs` and its subdomains
- `kubernetes.io/blog`
- The Kubernetes GitHub org (`github.com/kubernetes`, `github.com/kubernetes-sigs`)

You are **not** allowed to search the open internet, use Stack Overflow, use your own notes/cheat sheets, or open any other tab. This changes exam prep in a specific, practical way: you should practice **navigating the actual kubernetes.io docs quickly** (bookmark the pages you use most: Pod spec reference, `kubectl` cheat sheet page, RBAC, NetworkPolicy examples) rather than relying on external cheat sheets you've memorized from elsewhere — those won't exist on exam day, but the docs will, so knowing exactly where to find something in them fast is a real, practicable skill.

## Time management strategy

- **Skip and flag, don't get stuck.** If a task isn't clicking within roughly your allotted per-task budget, mark it (there's a flag/bookmark feature) and move on — you can return to flagged tasks later. A single 20-minute stall on one task can single-handedly cost you the exam if it eats time from three other tasks you'd have solved quickly.
- **Read the full task before typing anything.** Tasks often have multiple sub-requirements bundled into one description (e.g., "create a pod X with these labels, in this namespace, mounting this configmap, with this resource limit") — missing one sub-requirement after doing the rest correctly costs you partial-to-full credit for that entire task depending on grading.
- **Verify your context before every task.** `kubectl config current-context` (or read the task's provided context-switch command carefully) — solving a task correctly against the wrong cluster scores zero.
- **Verify your work before moving on**, but don't over-verify — a quick `kubectl get`/`kubectl describe` to confirm the object exists and looks right is worth the 15 seconds; re-reading the entire task three times over is not.
- **Know the weighting** (published in the exam curriculum — Workloads & Scheduling, Cluster Architecture/Installation/Configuration, Services & Networking, Storage, Troubleshooting are the graded domains, each with a percentage weight) so you can mentally allocate more careful attention to higher-weighted domains rather than treating every task as equally important blind guesswork.

## Points to Remember

- 17 tasks, 2 hours, unequal weighting, 66% to pass (confirm current numbers before your sitting — these details shift between exam versions).
- Only `kubernetes.io/docs`, `kubernetes.io/blog`, and the Kubernetes GitHub orgs are allowed in a single browser tab — no external search, no personal notes.
- Time management is a first-class exam skill: skip-and-flag stuck tasks rather than burning disproportionate time on one problem.
- Always confirm the correct context/cluster before acting on a task — correct work against the wrong cluster scores zero.
- Read every sub-requirement in a task description before starting — partial completion of a multi-part task usually costs partial-to-full credit.

## Common Mistakes

- Spending 20+ minutes perfecting one task while three easier, equally- or higher-weighted tasks go untouched due to running out of time.
- Forgetting to switch `kubectl context` at the start of a new task and confidently solving it against the previous task's cluster.
- Trying to use browser search or muscle-memory external references during the exam out of habit, risking a proctoring flag for navigating outside the allowed domains.
- Not reading a task's full description carefully and missing an embedded sub-requirement (a specific label, a specific namespace, a specific resource limit) that's easy to satisfy but easy to overlook when skimming under time pressure.
- Treating practice exams as "read the answer if stuck immediately" instead of genuinely timing yourself under real exam conditions — this creates a false sense of readiness that evaporates on the actual timed, proctored exam.
