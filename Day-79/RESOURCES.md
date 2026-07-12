# Day 79 — Resources: Runtime Security

## Primary (assigned)

- **Falco documentation** (falco.org/docs) — free, the assigned starting point for this day. Covers rule syntax, deployment modes (kernel module vs. eBPF probe), and the full default ruleset in depth.

## Deepen your understanding

- **eBPF.io** (ebpf.io) — the canonical, vendor-neutral explainer for what eBPF is and the class of problems (networking, observability, security) it solves; useful for grounding *why* Falco and Tetragon are built the way they are rather than treating eBPF as a black box.
- **Tetragon documentation** (tetragon.io/docs) — Cilium's eBPF-native security observability and enforcement project; read this specifically to understand the detect-vs-enforce distinction versus Falco.
- **Kubernetes documentation — Auditing** (kubernetes.io/docs/tasks/debug/debug-cluster/audit) — the official reference for audit policy stages, levels, and backend configuration (log vs. webhook).
- **Kubernetes documentation — Seccomp and AppArmor** (kubernetes.io/docs/tutorials/security) — the official tutorials for both `securityContext` fields, including the `Localhost` profile workflow end to end.

## Reference / lookup

- **Falco Rules GitHub repo** (github.com/falcosecurity/rules) — the actual source of the default and community rule sets; a good place to see real, production-tuned rule examples beyond the basics.
- **CNCF — Falco project page** (cncf.io) — background on Falco's graduation status and ecosystem (Falcosidekick, Falco Talon for automated response actions).

## Practice

- **KillerCoda / Katacoda-style free Kubernetes security scenarios** — several free interactive scenarios exist specifically for practicing Falco rule-writing and audit-log correlation without needing your own cluster infrastructure.
- Reuse the `kind` cluster from today's lab (audit policy + Falco already wired up) as an ongoing personal sandbox — deliberately break something (an overly broad RBAC role, a missing `NetworkPolicy`) and practice the full incident-response playbook from today's third README against a self-inflicted, low-stakes scenario.
