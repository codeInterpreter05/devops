# Day 120 — CKS Exam Prep: Exam Format & Curriculum Domains

**Phase:** 4 – Advanced/Specialization | **Week:** W21 | **Domain:** Advanced Security | **Flag:** 📌 Milestone

## Brief

The Certified Kubernetes Security Specialist (CKS) is the security-focused sibling to the CKA: same live, timed, remotely-proctored, performance-based format — but unlike CKA, **CKS requires an active CKA certification as a prerequisite**. That's not a formality; it reflects what the exam actually tests. CKS assumes you can already operate a cluster fluently (fast `kubectl`, YAML editing, static pod manifests, troubleshooting) and spends the entire two hours testing whether you can *secure* one under time pressure — hardening, detection, and response, not general administration. This day is the capstone of the Zero Trust/mTLS week: everything from SPIFFE/SPIRE, mTLS, and NetworkPolicy (Day 119) plus RBAC and admission control from earlier phases gets folded into exam-ready form alongside today's runtime-security and supply-chain topics.

This day is split into three files:

1. **This file** — exam format, curriculum domains, and CKS-specific strategy.
2. **[02-README-Runtime-Security-Falco-Seccomp-AppArmor-Audit.md](02-README-Runtime-Security-Falco-Seccomp-AppArmor-Audit.md)** — Kubernetes audit logging, Falco rule writing, seccomp and AppArmor.
3. **[03-README-Supply-Chain-Security-And-Cluster-Hardening.md](03-README-Supply-Chain-Security-And-Cluster-Hardening.md)** — kube-bench/CIS Benchmarks, Trivy Operator, image supply-chain enforcement, and NetworkPolicy as a hardening checklist item.

## Exam format — the concrete numbers (verify before your sitting)

- **Prerequisite:** an active CKA certification. You cannot register for CKS without one — this is the one Linux Foundation Kubernetes cert with a hard cert-of-cert prerequisite.
- **~2 hours**, live performance-based tasks against real clusters, delivered via the same PSI Secure Browser remote-proctoring setup as CKA (webcam-monitored, strict desk/room rules — re-check the current candidate handbook, since proctoring rules get updated).
- Historically **fewer, denser tasks than CKA** — CKS tasks tend to chain multiple security controls into one scenario rather than testing one isolated skill, so per-task time budgets are less uniform than "just divide total time by task count."
- **Passing score is in the mid-60s to high-60s percent range** — confirm the exact current number on the Linux Foundation/CNCF exam page before you register; like CKA, this threshold has shifted between exam versions.
- **Open-book**, restricted to specific documentation domains during the exam — `kubernetes.io/docs` and the Kubernetes GitHub orgs at minimum, historically also extended to cover a small set of security-tool docs relevant to the curriculum (Falco's docs, for example) — the exact allowed-domain list is published in the current candidate handbook and does change, so check it directly rather than assuming last year's list still applies.
- **Two free killer.sh CKS simulator sessions** bundled with every exam purchase, same 36-hours-once-activated model as the CKA simulator.

## Curriculum domains

The CNCF-published CKS curriculum groups tasks into six weighted domains (exact percentages have shifted slightly between curriculum revisions — always check the current published curriculum PDF before finalizing your study plan):

| Domain | Approx. weight | What it covers |
|---|---|---|
| Cluster Setup | ~10% | Secure cluster provisioning choices — network policies at setup time, CIS benchmark application, ingress/TLS setup, node metadata protection |
| Cluster Hardening | ~15% | RBAC least-privilege, restricting API server access, upgrading/patching, service account hardening |
| System Hardening | ~15% | Host OS attack surface reduction — kernel hardening, minimizing IAM/host access, restricting network access to kernel-critical processes |
| Minimize Microservice Vulnerabilities | ~20% | Pod Security Standards/admission, `securityContext` hardening, secrets management, mTLS between services |
| Supply Chain Security | ~20% | Image scanning, minimal/verified base images, static analysis of manifests, image signing and admission-time verification |
| Monitoring, Logging and Runtime Security | ~20% | Audit logging, Falco, behavioral analytics, immutability of containers at runtime |

Mapped to today's spreadsheet sub-topics: **Falco, seccomp, AppArmor, and audit logs** live squarely in *Monitoring, Logging and Runtime Security*; **Trivy Operator and image supply-chain controls** live in *Supply Chain Security*; **kube-bench and NetworkPolicy** span *Cluster Hardening* and *System Hardening*, since CIS benchmark remediation and network segmentation both reduce attack surface at the cluster and host level respectively.

## Strategy differences from CKA

- **Tasks chain concepts.** A CKA task might be "create this Deployment." A CKS task is more likely "this pod is running with an overly permissive security context — remove the offending capability, add a seccomp profile, and confirm Falco no longer flags the syscall it was making." Solving one layer and moving on likely leaves partial credit on the table.
- **Speed still matters, but the bottleneck shifts.** `kubectl` fluency (aliases, generate-then-edit, `--dry-run=client -o yaml`) from CKA prep is assumed baseline — the actual time sink in CKS tasks is *diagnosing* which specific misconfiguration a task is pointing at, not typing YAML.
- **Tool output IS the task, constantly.** A large share of realistic CKS-style scenarios hand you the output of `kube-bench`, `trivy image`, or a Falco alert and ask you to remediate what it found. Practicing *reading* that output fast and translating it directly into the right flag/manifest change matters as much as knowing the remediation itself — this is why today's labs are built around running these tools against real output, not just reading about what they do.
- **Static pod manifests reappear constantly.** Nearly every control-plane hardening task (audit logging, `--anonymous-auth`, `--insecure-port`) means editing `/etc/kubernetes/manifests/kube-apiserver.yaml` (or `etcd.yaml`/`kube-controller-manager.yaml`) directly and waiting for the kubelet to restart the static pod — there is no `kubectl apply` for these, and getting a volume mount wrong here fails silently (the apiserver just doesn't come back, with no `kubectl` command available to tell you why, since the API server itself is down).

## Points to Remember

- CKS requires an active CKA cert first — it assumes cluster-admin fluency and tests security specifically on top of it.
- Six curriculum domains: Cluster Setup, Cluster Hardening, System Hardening, Minimize Microservice Vulnerabilities, Supply Chain Security, Monitoring/Logging/Runtime Security — weights and exact task counts shift between curriculum versions, so verify current numbers before planning study time allocation.
- CKS tasks chain multiple controls together far more than CKA tasks do — partial completion of one layer in a multi-layer task usually isn't the full task.
- Tool output (kube-bench, trivy, Falco alerts) is frequently the task itself — practice interpreting output fast, not just running the tool.
- Control-plane hardening tasks mean editing static pod manifests directly on the node filesystem, not `kubectl apply` — a bad edit here can silently crash-loop the API server itself.

## Common Mistakes

- Attempting CKS prep before being genuinely fast at CKA-level `kubectl`/YAML tasks — CKS assumes that speed as a given and doesn't budget extra time for it.
- Memorizing tool names and one-line descriptions ("Falco does runtime detection") without practicing reading real tool output and translating it into an exact remediation command — this is where CKS time actually gets spent.
- Editing a static pod manifest to add a flag (e.g., an audit-log flag) without also adding the required `volumeMounts`/`volumes` entries for any file paths that flag references — causes a silent apiserver crash-loop with no `kubectl` feedback since the control plane itself is down.
- Assuming the allowed-documentation domain list and exact passing score are the same as when a friend took the exam last year — both have changed between CKS versions; always re-check the current candidate handbook before your sitting.
- Treating all six curriculum domains as equally worth drilling — under real time constraints, weighting study time toward the higher-percentage domains (Supply Chain Security, Minimize Microservice Vulnerabilities, Monitoring/Logging/Runtime Security) pays off more than spreading effort evenly.
