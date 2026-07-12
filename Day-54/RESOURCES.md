# Day 54 — Resources: CKA Exam Prep I

## Primary (assigned)

- **killer.sh CKA simulator** (killer.sh) — the assigned starting point; two free sessions are bundled with every official CKA exam purchase, and it's built by the same team that delivers the real proctored exam environment.

## Deepen your understanding

- **Linux Foundation / CNCF — CKA Candidate Handbook and Exam Curriculum** (training.linuxfoundation.org) — the authoritative source for current task count, timing, passing score, curriculum domain weightings, and proctoring rules; always check this directly before your actual exam date since these details do shift between exam versions.
- **Kubernetes docs — kubectl Cheat Sheet** (kubernetes.io/docs/reference/kubectl/cheatsheet) — one of the pages you're explicitly allowed to reference live during the exam; get familiar with its layout now so you can find things in it fast under time pressure.
- **"CKA Exam Series" by Kim Wüstkamp / KodeKloud CKA course** — widely used, hands-on-lab-heavy prep courses that closely track the real curriculum domains and include their own timed practice environments.
- **Kubernetes docs — Cluster Architecture, Workloads, Services & Networking, Storage sections** — the actual graded curriculum domains; skim each section's page structure (not just content) so you know exactly where to navigate during the exam itself.

## Reference / lookup

- `kubectl explain <resource>` — in-terminal, always-available field reference, faster than a docs search for simple structural questions.
- **`kind` documentation** (kind.sigs.k8s.io) — full config reference for multi-node clusters, custom networking, and loading local images, useful for building more elaborate practice scenarios later.

## Practice

- **`kind` locally** — free, disposable, near-instant multi-node clusters for unlimited timed drilling, as used in today's lab.
- **killer.sh (your two bundled sessions)** — the highest-fidelity practice available; sequence them deliberately (early baseline, late confirmation) rather than using both immediately.
- **KodeKloud CKA practice labs / Play with Kubernetes** — additional free/low-cost scenario-based practice environments if you want more repetition beyond `kind` and killer.sh.
