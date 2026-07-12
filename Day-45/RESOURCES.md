# Day 45 — Resources: K8s Advanced Scheduling

## Primary (assigned)

- **Kubernetes docs: Scheduling, Preemption, and Eviction** (kubernetes.io/docs/concepts/scheduling-eviction/) — free, the assigned starting point. Covers affinity, taints/tolerations, topology spread, priority/preemption, and eviction as one coherent section straight from upstream docs.

## Deepen your understanding

- **"Assigning Pods to Nodes" (Kubernetes docs)** — the dedicated deep page on nodeSelector and node/pod affinity syntax and semantics, including the OR/AND term rules that trip people up.
- **"Pod Topology Spread Constraints" (Kubernetes docs)** — the authoritative reference on `maxSkew`/`whenUnsatisfiable`, including multi-constraint examples.
- **kubernetes-sigs/descheduler GitHub repo** — README documents every built-in strategy with config examples; the fastest way to see what's actually available beyond the concepts in this day's notes.
- **"Pod Priority and Preemption" (Kubernetes docs)** — covers the PDB-vs-preemption interaction and the reserved system priority range explicitly.

## Reference / lookup

- `kubectl explain pod.spec.affinity` / `kubectl explain pod.spec.topologySpreadConstraints` — inline field-by-field schema reference directly from your cluster's API, always version-matched to what you're running.
- **Karpenter documentation — "Scheduling"** — how a modern node autoscaler interacts with taints, affinity, and topology spread when provisioning new nodes on demand.

## Practice

- **Killercoda / KillerShell Kubernetes scenarios** (killercoda.com) — free, browser-based Kubernetes sandboxes with several scenarios specifically on affinity, taints, and scheduling — good for repeating today's lab patterns without needing your own multi-node cluster.
