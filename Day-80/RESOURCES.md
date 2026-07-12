# Day 80 — Resources: Phase 2 Project — Full Pipeline

## Primary (assigned)

- **Your own notes** — the assigned resource for today is everything you've built across Phase 2 (Days 61–79): vulnerability scanning, GitOps/Argo CD, Kubernetes admission control, runtime security, DORA metrics. Today's job is synthesis and execution, not new material — treat every prior day's README/LAB as the actual reference set for this capstone.

## Deepen your understanding

- **Argo CD documentation — Image Updater** (argocd-image-updater.readthedocs.io) — the official reference for automating tag bumps from a registry, closing the loop between CI pushing an image and Argo CD deploying it without a manual manifest edit.
- **Kyverno Policy Library** (kyverno.io/policies) — a large catalog of real, production-grade policies (image verification, pod security, resource requirements) to adapt rather than writing every policy from scratch.
- **Sigstore / cosign documentation** (docs.sigstore.dev) — the authoritative reference for keyless and key-based image signing, and how verification integrates with admission controllers like Kyverno.
- **k6 documentation — Thresholds and Test Types** (grafana.com/docs/k6) — covers ramping-VU load patterns, threshold-based pass/fail gating, and how to structure a load test that's runnable in CI, not just manually.

## Reference / lookup

- **CNCF DevSecOps reference architectures / Trail of Bits & CNCF's "Software Supply Chain Best Practices"** — a solid checklist-style reference for what a complete supply-chain-secured pipeline (build, sign, verify, admit) should cover, useful for sanity-checking this project against an industry-recognized standard.
- **Mermaid documentation** (mermaid.js.org) — the full diagram syntax reference beyond the basic flowchart used in today's example.

## Practice

- Revisit **Day 60's portfolio-audit checklist** and run it against this specific capstone repo once finished — it's the same discipline (README quality, secret scanning, legible commit history, a stated tradeoff) applied to the single most important repo you'll produce in Phase 2.
- Share the finished, published repo with a peer or mentor and ask them to specifically try the "how do you reproduce this" and "where exactly is X enforced" questions against your README — a real reader's confusion is the most valuable feedback signal available before an actual interview does the same thing.
