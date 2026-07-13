# Day 115 — Kubernetes Cost Optimisation: Spot Nodes with Karpenter

**Phase:** 4 – Advanced/Specialization | **Week:** W20 | **Domain:** FinOps | **Flag:** —

## Brief

Day 114 covered Spot strategy at the EC2 level; Karpenter is the tool that makes Spot (and right-sized nodes generally) actually practical inside Kubernetes, because it replaces the rigid "pre-defined node groups + Cluster Autoscaler" model with just-in-time node provisioning shaped to whatever's actually pending in the scheduler queue. This is one of the highest-leverage cost tools in the modern EKS ecosystem and shows up constantly in cost-optimization interview questions.

## Why Karpenter exists: the Cluster Autoscaler's limitation

The traditional model is: define fixed EC2 Auto Scaling Groups (ASGs) per instance type/size (a "node group"), and let **Cluster Autoscaler** add/remove nodes *within* those pre-defined groups when pods are pending or nodes are idle. The problem: you, a human, pre-decided the shapes of nodes available. If a pod needs 7 vCPUs and your node groups only offer 4 or 8 vCPU instances, you get an 8-vCPU node with 1 vCPU wasted — every time, for every pod like that.

**Karpenter** removes the node-group abstraction entirely. It watches for **unschedulable pods**, looks at their actual resource requests, and asks the cloud provider's API directly for the best-fitting instance type/size **at that moment** — no pre-defined groups needed. It can also proactively **consolidate**: continuously re-evaluate whether existing nodes could be replaced by fewer/cheaper/better-fitting nodes and, if so, safely drain and replace them.

## Configuring Karpenter for Spot

A **NodePool** (Karpenter's CRD for "what kinds of nodes am I allowed to create") defines constraints; an **EC2NodeClass** defines the cloud-specific details (AMI, subnet, security group, instance profile):

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: spot-general
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]     # allow both; Karpenter picks cheapest fit
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: node.kubernetes.io/instance-type
          operator: In
          values: ["m5.large","m5.xlarge","m6i.large","m6i.xlarge","c5.large","c5.xlarge"]
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 30s
  limits:
    cpu: "1000"
```

Key design choices here, and why they matter:

- **Listing multiple instance types/families** (`m5`, `m6i`, `c5`, various sizes) isn't just flexibility for its own sake — it directly reduces Spot interruption risk by spreading the fleet across many independent Spot capacity pools, exactly as covered in Day 114. A NodePool restricted to one instance type defeats this.
- **`capacity-type: In [spot, on-demand]`** lets Karpenter fall back to On-Demand automatically if Spot capacity isn't available for any allowed type at that moment, rather than leaving pods pending — this is the single biggest reason Karpenter+Spot is safer than manually managing separate Spot/On-Demand ASGs, because the fallback is automatic and instant.
- **`consolidationPolicy: WhenEmptyOrUnderutilized`** is what actually captures bin-packing savings over time: Karpenter periodically checks if it could replace a set of underutilized nodes with fewer/cheaper ones and does so safely (respecting PodDisruptionBudgets), continuously fighting the utilization decay that happens as workloads scale up and back down over time.

**Isolating what must never land on Spot** — use a separate NodePool with taints, and have critical workloads tolerate only that pool:

```yaml
# on-demand-critical NodePool
spec:
  template:
    spec:
      taints:
        - key: dedicated
          value: on-demand-critical
          effect: NoSchedule
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand"]
---
# In the critical Deployment's pod spec:
tolerations:
  - key: dedicated
    value: on-demand-critical
    operator: Equal
    effect: NoSchedule
```

## Handling Spot interruption gracefully

Karpenter integrates with the Spot **interruption notice** (via SQS + EventBridge rule watching for `EC2 Spot Interruption Warning` and rebalance recommendation events) — when configured, it proactively cordons and starts draining a node the moment AWS signals an upcoming reclaim, rather than waiting for the hard 2-minute termination:

```yaml
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  # ... AMI, subnet, etc.
  # Interruption handling requires an SQS queue + EventBridge rules set up
  # alongside the Karpenter controller's IAM permissions (see Karpenter docs'
  # "getting started" for the exact CloudFormation/Terraform for this queue).
```

Without this wiring, a Spot node still gets reclaimed on schedule, but Kubernetes only finds out when the node abruptly disappears (kubelet stops reporting), which is a harder, less graceful recovery than a proactive cordon-and-drain.

## Points to Remember

- Karpenter provisions nodes directly from the cloud API based on actual pending-pod requirements — it doesn't use pre-defined ASG node groups the way Cluster Autoscaler does, which is what lets it fit node shapes tightly to workload shapes.
- Allow multiple instance types/families in a NodePool, not just multiple AZs — Spot capacity pools are per-type-per-AZ, and diversity across both axes is what reduces interruption risk.
- `capacity-type: In [spot, on-demand]` gives automatic On-Demand fallback when Spot capacity is unavailable — this removes most of the operational risk of running Spot in production.
- `consolidationPolicy` is what continuously fights bin-packing decay — without it, a cluster's efficiency degrades over time as workloads scale up and down and leave underutilized nodes behind.
- Wiring up the SQS/EventBridge Spot interruption notice path lets Karpenter cordon-and-drain proactively instead of waiting for the hard 2-minute termination — a meaningfully more graceful failure mode.

## Common Mistakes

- Creating a Karpenter NodePool restricted to a single instance type "to keep things simple" — this reintroduces the exact single-pool Spot interruption risk that diversification is meant to solve.
- Forgetting to set `limits` on a NodePool, letting Karpenter scale unbounded in response to a pending-pod spike (e.g., a runaway job or a misconfigured HPA) — always cap total CPU/memory a NodePool can provision.
- Running critical, stateful, or singleton workloads on a NodePool that allows Spot without a taint/toleration scheme to keep them off it — assuming "Karpenter will just fall back to On-Demand" is not a substitute for explicitly excluding workloads that truly cannot tolerate any interruption.
- Not setting up the Spot interruption notice SQS/EventBridge integration, then being surprised that node replacement under Spot reclaim looks "less graceful" than expected — the graceful path is opt-in infrastructure, not automatic.
- Setting `consolidateAfter` unrealistically low for spiky workloads, causing Karpenter to consolidate nodes away right before a traffic spike returns, leading to thrashing (constant node creation/deletion) instead of savings.
