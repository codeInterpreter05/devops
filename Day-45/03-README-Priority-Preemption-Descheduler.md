# Day 45 — K8s Advanced Scheduling: Priority Classes, Preemption & Descheduler

**Phase:** 1 – Core DevOps | **Week:** W7 | **Domain:** Kubernetes | **Flag:** –

## Brief

Affinity, taints, and topology spread constraints (previous two files) all answer "where should this pod go, at the moment it's scheduled?" This file covers what happens **under resource pressure** — when the cluster is full and something has to give — and how Kubernetes corrects for scheduling decisions that were fine when made but have become suboptimal over time (nodes drained and refilled unevenly, new nodes added after initial placement, etc).

## PriorityClasses

A `PriorityClass` is a cluster-scoped object assigning an integer priority value to pods that reference it. Higher number = higher priority.

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: business-critical
value: 1000000
globalDefault: false
description: "Reserved for revenue-critical services"
```

```yaml
spec:
  priorityClassName: business-critical
```

- **`globalDefault: true`** on at most one PriorityClass sets the priority applied to any pod that doesn't specify one — if you never set this, unspecified pods get priority `0`, which is *lower* than most sensible custom classes, meaning by default, everything you didn't explicitly prioritize is preemptible by anything you did.
- Kubernetes reserves a range of very high values (≥ 1 billion) for **system-critical** priority classes (`system-cluster-critical`, `system-node-critical`, used by core components like CoreDNS and kube-proxy) — don't try to out-prioritize these; they're intentionally above anything a normal workload should claim.
- Priority affects two distinct things: **scheduling order** (higher-priority pending pods are considered by the scheduler before lower-priority ones when there's a queue) and **preemption eligibility** (see below).

## Preemption

When a high-priority pod can't be scheduled because the cluster lacks capacity, the scheduler can **preempt** (evict) lower-priority pods to make room — but only as a last resort, and only pods with strictly lower priority are eligible victims.

**Mechanics:**
1. Scheduler tries to place the pending pod normally; fails due to insufficient resources anywhere.
2. Scheduler looks for a node where evicting one or more lower-priority pods would free enough capacity.
3. If found, it **nominates** that node (visible via `pod.status.nominatedNodeName`), then triggers graceful termination of the victim pod(s) (respecting their `terminationGracePeriodSeconds`).
4. The preempting pod is *not* immediately bound to that node — it goes back through normal scheduling once resources free up, which occasionally means it lands somewhere else if conditions changed in the meantime.

**PodDisruptionBudgets (PDBs) do not block preemption** — this is a frequently misunderstood and interview-relevant point. PDBs protect against *voluntary* disruptions (node drains, cluster-autoscaler scale-down, `kubectl drain`) but preemption is treated as a **necessary** action to schedule a higher-priority pod, so the scheduler can and will violate a PDB's minimum-available guarantee if that's what it takes. This is a deliberate design decision (a pending critical pod is considered a worse outcome than briefly violating a PDB) but it surprises people who assumed PDBs were an absolute floor.

**Preemption doesn't guarantee success** — if evicting lower-priority pods still wouldn't free enough resources (e.g., due to node affinity/taint constraints that also apply to the pending pod), preemption won't happen just to "try," and the pod stays `Pending`.

## Descheduler

The default scheduler only makes placement **decisions at pod creation time** — it never revisits an already-running pod's placement, even if the cluster's state has since made that placement suboptimal (a node that was full when a pod landed is now half-empty; new, better-fitting nodes joined afterward; taints/affinity rules changed). The **Descheduler** (a separate, optional component, `kubernetes-sigs/descheduler`) fills this gap by periodically evaluating running pods against a set of configurable strategies and evicting ones that violate them — relying on the scheduler to then replace them in a better spot.

**Common built-in strategies:**
- `RemoveDuplicates` — evict extra replicas of the same ReplicaSet/Job piled onto one node.
- `LowNodeUtilization` — evict pods from over-utilized nodes to rebalance toward under-utilized ones.
- `RemovePodsViolatingNodeAffinity` / `RemovePodsViolatingNodeTaints` — evict pods whose node no longer satisfies affinity rules or now carries a taint the pod doesn't tolerate (recall from file 01 that affinity/taints are normally `IgnoredDuringExecution` — the descheduler is the mechanism that actually enforces ongoing compliance if you want it).
- `RemovePodsViolatingTopologySpreadConstraint` — re-balance when topology spread has drifted (e.g., after a node was replaced and pods piled unevenly).
- `RemovePodsViolatingInterPodAntiAffinity` — similar, for pod anti-affinity drift.

Runs as either a `CronJob` (periodic batch run) or a long-running `Deployment` (continuous, event-driven in newer versions) inside the cluster, using its own RBAC permissions to evict pods via the standard Eviction API (respecting PDBs, unlike preemption — descheduler evictions **are** voluntary disruptions and do respect PDB limits).

**Why this matters in practice**: without a descheduler, a cluster's pod placement quality only ever gets *set* at creation time and *degrades* over time as nodes come and go (especially true with Cluster Autoscaler/Karpenter constantly adding/removing nodes) — the descheduler is what keeps actual placement honest against your declared affinity/spread/taint policies on an ongoing basis, rather than a one-time best effort.

## Points to Remember

- Priority is an integer; higher wins. Set exactly one `globalDefault: true` PriorityClass (or accept that unspecified pods default to priority 0, the lowest).
- Preemption evicts strictly lower-priority pods to make room for a pending higher-priority one — and it explicitly **can violate PodDisruptionBudgets**, unlike voluntary disruptions.
- Preemption doesn't guarantee the preempting pod lands on the node it preempted from — it re-enters normal scheduling afterward.
- The default scheduler never revisits already-placed pods — the Descheduler is the separate component that corrects drift (uneven utilization, stale affinity/taint compliance, topology spread violations) on already-running pods.
- Descheduler evictions are voluntary disruptions and do respect PDBs — the opposite of preemption in that specific regard.

## Common Mistakes

- Assuming a PodDisruptionBudget protects a workload from ever losing pods unexpectedly — it protects against voluntary disruptions, not against preemption by a higher-priority pending pod.
- Not setting a `globalDefault` PriorityClass, then being surprised that a wide variety of unrelated, unprioritized workloads are all equally (and lowly) preemptible by anything explicitly prioritized.
- Assigning arbitrarily huge priority values to "important" application workloads that creep close to or exceed the reserved system-critical range, risking confusing interactions with core cluster components.
- Deploying the Descheduler with aggressive strategies (like `LowNodeUtilization`) without PDBs properly configured on the affected workloads, causing unnecessary churn/evictions on services that can't tolerate it.
- Believing that node affinity/taint rules are continuously enforced on running pods by default — they're only checked at scheduling time; only the Descheduler's specific strategies (`RemovePodsViolatingNodeAffinity`/`Taints`) actually evict pods that have drifted out of compliance.
