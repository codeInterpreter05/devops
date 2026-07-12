# Day 45 — K8s Advanced Scheduling: nodeSelector, Node Affinity & Taints/Tolerations

**Phase:** 1 – Core DevOps | **Week:** W7 | **Domain:** Kubernetes | **Flag:** –

## Brief

The Kubernetes scheduler's default behavior — "find any node with enough free CPU/memory" — is fine for a toy cluster and actively wrong for production. Real clusters mix instance types (spot vs. on-demand, GPU vs. CPU-only), span multiple AZs for resilience, and need certain workloads kept away from certain nodes (or certain nodes reserved for certain workloads). `nodeSelector`, node affinity, and taints/tolerations are the three mechanisms that give you that control, and they show up constantly in interviews because "how do you make sure critical pods never land on spot nodes" (this day's interview question) has a precise, correct answer that only taints/tolerations actually provide.

This day is split into three files:

1. **This file** — `nodeSelector`, node affinity, and taints/tolerations (control over *which node a pod can/must run on*).
2. **[02-README-Pod-Affinity-Topology-Spread.md](02-README-Pod-Affinity-Topology-Spread.md)** — pod affinity/anti-affinity and topology spread constraints (control over *how pods are placed relative to each other*).
3. **[03-README-Priority-Preemption-Descheduler.md](03-README-Priority-Preemption-Descheduler.md)** — priority classes, preemption, and the descheduler (control over *what happens under resource pressure, after initial scheduling*).

## `nodeSelector` — the blunt instrument

The simplest mechanism: a flat key-value match against node labels.

```yaml
spec:
  nodeSelector:
    disktype: ssd
```
The pod will **only** be scheduled onto nodes labeled `disktype=ssd`; if no such node exists or has capacity, the pod stays `Pending`. It's an exact-match AND — multiple keys must all match. No OR logic, no "prefer but don't require," no operators like `In`/`NotIn`. This is why node affinity superseded it for anything beyond trivial cases — `nodeSelector` still works and is still valid, but it's essentially the deprecated-in-spirit, simple subset of node affinity.

## Node affinity — expressive, two-strength rules

Node affinity generalizes `nodeSelector` with **operators** (`In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt`) and, crucially, **two strength levels**:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 80
      preference:
        matchExpressions:
        - key: node-lifecycle
          operator: In
          values: ["on-demand"]
```

- **`required...`** (hard constraint): the scheduler will **not** place the pod on a node that doesn't satisfy this — functionally like a smarter `nodeSelector`. If unsatisfiable, the pod stays `Pending`.
- **`preferred...`** (soft constraint): the scheduler tries to honor it, weighted (1-100) against other preferences, but will still schedule the pod elsewhere if no matching node has capacity. Use this for "ideally here, but don't block scheduling if not possible" — e.g., preferring on-demand nodes without hard-failing when only spot capacity remains.
- **`IgnoredDuringExecution`** (present in both, and currently the only supported behavior): once a pod is running, if the node's labels change such that it no longer matches, **the pod is not evicted**. Affinity is evaluated only at scheduling time. (A `RequiredDuringSchedulingRequiredDuringExecution` variant that *would* evict on drift has been discussed for years but is not GA — don't claim it exists.)

**Multiple `nodeSelectorTerms` are OR'd together; multiple `matchExpressions` within one term are AND'd** — this is a common point of confusion when writing more complex rules with several terms.

## Taints and tolerations — repel, don't attract

This is the piece people conflate with affinity, and the distinction is the crux of this day's interview question. **Affinity is about what a pod wants** (attraction toward nodes). **Taints are about what a node refuses** (repulsion of pods) — and a **toleration** on a pod is permission to ignore a specific taint, not an instruction to prefer that node.

```bash
kubectl taint nodes gpu-node-1 workload=gpu:NoSchedule
kubectl taint nodes spot-node-1 lifecycle=spot:NoSchedule
```

```yaml
tolerations:
- key: "lifecycle"
  operator: "Equal"
  value: "spot"
  effect: "NoSchedule"
```

**Three taint effects:**
- `NoSchedule` — new pods without a matching toleration won't be scheduled here; existing pods already running are unaffected.
- `PreferNoSchedule` — soft version; the scheduler tries to avoid it but will place pods here if needed.
- `NoExecute` — the aggressive one: pods without a matching toleration are **evicted if already running**, not just blocked from new scheduling. Node conditions like `node.kubernetes.io/not-ready` and `node.kubernetes.io/unreachable` are auto-applied as `NoExecute` taints, which is *how* Kubernetes evicts pods from a node that's gone unresponsive (with a default `tolerationSeconds` grace period pods implicitly tolerate before eviction).

**Answering this day's interview question — "how do you ensure critical pods never land on spot/preemptible nodes":** the correct pattern is **taint the spot nodes**, not "add affinity to on-demand nodes." Taint every spot node (e.g., `lifecycle=spot:NoSchedule`), and only add a matching **toleration** to the pods that are explicitly allowed to run on spot (your fault-tolerant, interruption-safe batch workloads). Critical pods, having no toleration for that taint, are structurally incapable of landing there — the scheduler enforces it, not a convention or a label you hope stays consistent. If you tried to solve this with *only* node affinity ("prefer on-demand"), a `preferred` rule doesn't guarantee anything, and a `required` rule pinned to on-demand nodes only solves it for pods you remembered to configure — new pods without that affinity rule could still land on spot. Taints put the burden of opt-in on the exception (spot-tolerant workloads), not on every critical workload remembering to opt out.

In practice, cloud-managed spot node groups (EKS managed node groups, Karpenter) usually **auto-apply** a standard taint like `karpenter.sh/capacity-type=spot:NoSchedule` or `eks.amazonaws.com/capacityType=SPOT:NoSchedule` on spot nodes for exactly this reason — check whether your node provisioner already does this before hand-rolling your own taint key.

## Points to Remember

- `nodeSelector` is exact-match-only and effectively a legacy subset of node affinity — prefer affinity for anything beyond the simplest case.
- Node affinity has two strengths: `required` (hard, can leave pods Pending) and `preferred` (soft, weighted, never blocks scheduling).
- Affinity/anti-affinity is evaluated only `...DuringScheduling`, not continuously — a node's labels drifting after a pod starts running does not evict it (`IgnoredDuringExecution`).
- Taints repel; tolerations permit (not attract). A toleration lets a pod land on a tainted node — it doesn't make the scheduler prefer that node.
- `NoExecute` taints can evict already-running pods (with a `tolerationSeconds` grace period); `NoSchedule`/`PreferNoSchedule` only affect new scheduling decisions.
- To guarantee critical workloads never land on spot capacity: taint the spot nodes and add tolerations only to the workloads explicitly allowed there — don't rely on affinity alone for a hard guarantee.

## Common Mistakes

- Confusing tolerations with a scheduling preference — adding a toleration to a pod and assuming it now prefers the tainted node, when it only means the pod is *permitted* to land there among other candidates.
- Relying solely on `preferredDuringSchedulingIgnoredDuringExecution` node affinity to keep critical workloads off spot nodes, not realizing "preferred" is explicitly a best-effort hint that gets overridden under capacity pressure.
- Assuming affinity rules are continuously enforced and pods get evicted if a node's labels change — they don't, because affinity only applies `...DuringScheduling`.
- Writing multiple `matchExpressions` inside a single `nodeSelectorTerms` entry expecting OR semantics, when expressions within one term are AND'd (you need separate `nodeSelectorTerms` entries for OR logic).
- Forgetting that a managed node group or Karpenter provisioner may already auto-taint spot nodes with its own key, then adding a redundant/conflicting custom taint that makes debugging "why won't this pod schedule" unnecessarily confusing.
