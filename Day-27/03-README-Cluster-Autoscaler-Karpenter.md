# Day 27 — Autoscaling: Cluster Autoscaler vs Karpenter

**Phase:** 1 – Core DevOps | **Week:** W4 | **Domain:** Kubernetes | **Flag:** ⚡ Interview-critical

## Brief

HPA, VPA, and KEDA (previous files) all operate on **pods** — they assume nodes with enough capacity already exist. Someone still has to make sure the underlying **nodes** exist to actually schedule those pods onto. That's a different autoscaling problem entirely, solved at the infrastructure layer by the **Cluster Autoscaler** (the long-standing, cloud-agnostic-ish tool) or **Karpenter** (AWS's newer, faster, more flexible replacement) — and understanding the architectural difference between the two is now a standard, expected interview topic for anyone claiming EKS experience.

## The problem: pending pods, no capacity

When the scheduler can't place a pod anywhere (`FailedScheduling`, `Insufficient cpu`/`memory` in events — see Day 22), something needs to notice that and provision more node capacity. Neither HPA/VPA/KEDA do this — they only ask for more pods; getting those pods actual nodes to run on is the node-autoscaler's job.

## Cluster Autoscaler: works in terms of node groups

Cluster Autoscaler (CA) is the original, widely-used tool, and it operates around the concept of pre-defined **node groups / Auto Scaling Groups (ASGs)** on AWS:

1. It watches for unschedulable pods.
2. It simulates whether adding a node from one of its configured ASGs would let the pending pod schedule.
3. It calls the ASG's `SetDesiredCapacity` API to scale that ASG up by the needed number of nodes.
4. On scale-down, it looks for **underutilized nodes** (below a configurable threshold) whose pods could be safely rescheduled elsewhere, cordons and drains them, then scales the ASG back down.

```yaml
# Simplified: CA is configured with a list of ASGs it's allowed to manage
--nodes=1:10:my-eks-nodegroup-general
--nodes=0:5:my-eks-nodegroup-gpu
```

**The core limitation:** CA can only choose *which pre-defined ASG/node group* to scale — it cannot pick an arbitrary instance type/size on the fly. If your pending pod needs an instance shape that doesn't match any existing node group (e.g., you need more memory-per-CPU than any configured group offers), CA simply can't help; you must have pre-provisioned a matching node group in advance. This means real-world CA usage often involves maintaining a sprawling set of node groups (general-purpose, memory-optimized, GPU, spot vs on-demand, per-AZ) just to cover the space of workload shapes you might need.

```bash
kubectl get pods -n kube-system -l app=cluster-autoscaler
kubectl logs -n kube-system -l app=cluster-autoscaler | grep -i "scale-up"
kubectl describe configmap cluster-autoscaler-status -n kube-system   # CA's own health/decision summary
```

## Karpenter: works directly against the cloud API, no node groups

Karpenter (AWS-built, now graduated and widely adopted, also usable on other clouds via provider plugins) throws out the node-group abstraction entirely. Instead of picking from pre-defined ASGs, Karpenter:

1. Watches for unschedulable pods directly.
2. Looks at the *exact* resource requests, node selectors, tolerations, and topology spread constraints of the pending pod(s).
3. Computes the **cheapest, best-fit instance type and size** from the *entire* set of instance types allowed by its `NodePool`/`EC2NodeClass` configuration — potentially a completely different instance type for every scale-up event, chosen dynamically.
4. Calls the EC2 API **directly** to launch exactly that instance — no ASG involved at all.
5. On consolidation (Karpenter's term for scale-down/bin-packing), it continuously looks for opportunities to replace multiple underutilized nodes with fewer, better-packed ones, or terminate genuinely empty nodes — more proactive and granular than CA's threshold-based approach.

```yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata: { name: general-purpose }
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
      nodeClassRef: { name: default }
  limits:
    cpu: 1000
  disruption:
    consolidationPolicy: WhenUnderutilized
    consolidateAfter: 30s
```

Because Karpenter isn't constrained to pre-defined node group shapes, it typically achieves **faster provisioning** (launches instances directly, skipping ASG-level orchestration overhead) and **better bin-packing/cost efficiency** (picks the exact right-sized instance per workload rather than the closest pre-defined group), which is why most new EKS deployments default to Karpenter today, with Cluster Autoscaler considered more of a legacy/still-supported option for existing setups or non-AWS clusters.

## Cooldown periods and flapping prevention (applies across HPA/KEDA/CA/Karpenter alike)

"Flapping" — rapidly scaling up then down then up again — wastes resources on pod/node churn and can cause real user-facing instability (constant pod evictions, cold-start latency repeatedly paid). Every layer of autoscaling has its own cooldown/stabilization mechanism, and they need to be tuned coherently, not independently:

- **HPA**: `behavior.scaleDown.stabilizationWindowSeconds` (default 300s) — requires sustained low utilization before scaling down; `scaleUp` stabilization is typically much shorter or zero, since reacting fast to *increased* load is usually desired, while reacting fast to *decreased* load risks premature scale-down on a brief lull.
- **KEDA**: `cooldownPeriod` — same idea, specifically governing the return to `minReplicaCount`.
- **Cluster Autoscaler**: `--scale-down-unneeded-time` (default 10m) — a node must be underutilized continuously for this long before CA considers removing it.
- **Karpenter**: `consolidateAfter` — similar cooldown before consolidating/terminating underutilized nodes.

The general principle: **scale up fast, scale down slow.** Being briefly short on capacity under a load spike is usually worse (dropped requests, latency spikes) than briefly running extra capacity a bit longer than strictly necessary (wasted spend, but no user impact) — so cooldowns are almost always asymmetric by design, not a bug.

## Points to Remember

- Pod-level autoscalers (HPA/VPA/KEDA) and node-level autoscalers (Cluster Autoscaler/Karpenter) solve different problems and must both be present for a cluster to genuinely autoscale end-to-end — one requests more pods, the other ensures nodes exist to run them.
- Cluster Autoscaler scales pre-defined ASGs/node groups up or down — it cannot choose an instance type outside what you've already configured as a node group.
- Karpenter provisions nodes directly against the cloud API with no node-group concept, dynamically picking the best-fit instance type per pending pod — generally faster and more cost-efficient, and the modern default choice on EKS.
- All autoscaling layers need cooldown/stabilization tuning, and the general design principle everywhere is asymmetric: react fast to scale-up signals, react slow (require sustained evidence) to scale-down signals, to avoid flapping.
- Autoscaling pods without autoscaling nodes (or vice versa) is incomplete — HPA can decide "we need 20 replicas" but if there's no node capacity and no cluster/node autoscaler running, those extra pods just sit `Pending` forever.

## Common Mistakes

- Running HPA/KEDA without any cluster/node autoscaler at all, then being confused why scaled-up replicas sit `Pending` under load — pod-level autoscaling and node-level autoscaling are separate systems that must both be configured.
- Maintaining Cluster Autoscaler with a narrow, incomplete set of node groups, so certain pending pod shapes (unusual memory/CPU ratios, specific instance families) can never be satisfied no matter how long CA waits — a symptom of the "must pre-define every shape you might need" limitation.
- Assuming Karpenter and Cluster Autoscaler can run simultaneously managing the same nodes without conflict — they should generally be treated as mutually exclusive choices for a given node pool, not layered on top of each other.
- Setting scale-down cooldowns too aggressively low (fast scale-down) "to save cost," causing flapping — nodes/pods get removed just before the next load spike arrives, forcing an immediate, costlier-in-latency-terms scale back up.
- Ignoring Pod Disruption Budgets when relying on Cluster Autoscaler/Karpenter's scale-down (which cordons and drains nodes) — without a PDB, a scale-down event can evict enough replicas of a service simultaneously to cause a real availability dip, precisely the kind of thing PDBs exist to prevent.
