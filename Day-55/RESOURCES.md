# Day 55 — Resources: CKA Exam Prep II

## Primary (assigned)

- **Mumshad Mannambeth — Certified Kubernetes Administrator (CKA) with Practice Tests** (Udemy, KodeKloud) — the assigned starting point. Widely considered the gold-standard CKA course; includes the KodeKloud browser-based labs that mirror the exam's actual environment.

## Deepen your understanding

- **Kubernetes docs — Operating etcd clusters for Kubernetes** (kubernetes.io) — the authoritative source for `etcdctl snapshot save/restore` semantics and multi-member etcd operations.
- **Kubernetes docs — Upgrading kubeadm clusters** (kubernetes.io) — the exact, version-specific command sequence; always cross-check against whatever version you're studying, since flags occasionally shift between releases.
- **Kubernetes docs — Network Policies** (kubernetes.io) — the canonical explanation of default-allow/default-deny semantics with worked YAML examples for ingress, egress, and combined rules.
- **Kubernetes docs — Using RBAC Authorization** (kubernetes.io) — the full reference for Role/ClusterRole/RoleBinding/ClusterRoleBinding, including aggregated ClusterRoles (a more advanced pattern worth knowing exists even if not exam-core).

## Reference / lookup

- **killer.sh** — the official CKA exam simulator (comes bundled with your exam registration, two free sessions) — the closest thing to the real test environment and time pressure; do this close to your actual exam date.
- **kubernetes.io "Debug Running Pods" / "Troubleshoot Clusters"** — the official troubleshooting guide covering the `crictl`/`journalctl` workflow for when the API server itself is unavailable.

## Practice

- **KodeKloud CKA Ultimate Mock Exams** — timed, scored practice exams matching the real exam's task distribution (etcd, upgrades, RBAC, NetworkPolicy, troubleshooting all represented).
- Build a throwaway 2-node kubeadm cluster (Vagrant, Multipass, or cheap cloud VMs) specifically so you can safely practice the **upgrade** and **etcd restore** tasks — these are the two categories `kind`-only practice can't fully replicate.
