# Day 53 — Phase 1 Project: Auto-scaling with KEDA & Karpenter

**Phase:** 1 – Core DevOps | **Week:** W8 | **Domain:** Review | **Flag:** 📌 Milestone project

## Brief

The final piece of a production EKS environment is making it actually elastic — scaling pods to match real demand (KEDA) and scaling the underlying nodes to fit those pods efficiently (Karpenter). These solve two different layers of the same problem and are frequently confused with each other (and with the older Cluster Autoscaler / HPA) in interviews — being able to draw the clean line between "this scales pods" and "this scales nodes" is the actual signal being tested.

## KEDA — event-driven pod autoscaling

The built-in Horizontal Pod Autoscaler (HPA) scales pods based on **metrics-server-reported CPU/memory** (or custom metrics via the Metrics API, which requires standing up an adapter yourself). **KEDA (Kubernetes Event-Driven Autoscaling)** extends this dramatically: it can scale a Deployment based on **external event sources** — SQS queue depth, Kafka consumer lag, a Prometheus query result, a cron schedule, or dozens of other "scalers" — and, critically, it can scale a workload **to and from zero**, which the standard HPA cannot do at all (HPA's minimum is always 1 replica).

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-worker-scaler
  namespace: production
spec:
  scaleTargetRef:
    name: order-worker            # the Deployment being scaled
  minReplicaCount: 0               # scale to zero when idle — HPA cannot do this
  maxReplicaCount: 20
  cooldownPeriod: 300               # seconds of no events before scaling back to minReplicaCount
  triggers:
    - type: aws-sqs-queue
      metadata:
        queueURL: https://sqs.us-east-1.amazonaws.com/123456789012/order-events
        queueLength: "5"           # target ~5 messages per replica
        awsRegion: us-east-1
      authenticationRef:
        name: keda-aws-credentials  # IRSA-backed, no static keys
```

**Mechanically, KEDA works by creating and managing an HPA object under the hood** — it's not a replacement for HPA so much as a metrics source adapter plus a scale-to-zero controller sitting in front of it: KEDA's own operator polls the external source (SQS queue depth, in this example), translates it into a metric HPA can consume, and lets HPA do the actual scaling math above 0 replicas — while KEDA itself handles the 0-to-1 (and 1-to-0) transition that HPA structurally can't.

**Why this matters for this project's architecture**: a worker consuming from the SQS queue built on Day 52 is the textbook KEDA use case — during idle periods there's no reason to keep even one replica running (scale to zero, save cost), and during a burst of orders it should scale out proportionally to queue depth rather than waiting for CPU usage to climb (which only happens *after* messages are already backing up, a lagging indicator compared to queue depth itself).

## Karpenter — node-level autoscaling

Where KEDA (and HPA) decide **how many pods** you need, **Karpenter** decides **what nodes** those pods run on — and it's the modern replacement for the older **Cluster Autoscaler**, with a fundamentally different approach.

Cluster Autoscaler works within pre-defined node groups/Auto Scaling Groups, scaling the *count* of instances within a group whose instance type is fixed ahead of time — if pending pods need an instance type/size the existing node groups don't offer, Cluster Autoscaler simply can't help; you'd need to have pre-provisioned a node group for every shape of workload you might run.

**Karpenter observes unschedulable pods directly and provisions exactly-fitting nodes on demand**, without needing pre-defined node groups/ASGs at all — it looks at the actual pending pods' resource requests, taints/tolerations, and node affinity/selector requirements, then picks the cheapest instance type/size (and Spot vs. On-Demand, if you've allowed Spot) that fits, and provisions it directly via EC2 APIs.

```yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: general-purpose
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
      nodeClassRef:
        name: default
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized   # actively bin-packs and removes underused nodes
    expireAfter: 720h
---
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: AL2023
  subnetSelectorTerms:
    - tags: { "kubernetes.io/cluster/prod-eks-cluster": "shared" }
  securityGroupSelectorTerms:
    - tags: { "kubernetes.io/cluster/prod-eks-cluster": "shared" }
```

**Why Karpenter's bin-packing/consolidation matters for cost**: beyond just scaling up when pods are pending, Karpenter continuously evaluates whether existing nodes are underutilized and can be **consolidated** — draining and terminating a node whose remaining pods could be repacked onto fewer, better-utilized nodes elsewhere. Cluster Autoscaler does something similar but is constrained to scaling within existing node groups' instance types; Karpenter can actively pick a *different, better-fitting* instance type on consolidation, not just a different count of the same type.

## How KEDA and Karpenter work together (the full elastic picture)

1. A burst of SQS messages arrives (built in Day 52's project).
2. KEDA's SQS scaler notices queue depth exceeding the target and scales the `order-worker` Deployment from 0 to, say, 15 replicas via the HPA it manages under the hood.
3. Those 15 new pods can't be scheduled — no existing node has enough free capacity (this is the standard "Pending" pod state you'd see in `kubectl get pods`).
4. Karpenter observes the unschedulable pods, calculates the cheapest instance type/count that fits their combined requests, and provisions new EC2 nodes directly (often within under a minute, notably faster than traditional ASG-based scaling since Karpenter skips the ASG launch-template/scaling-policy indirection).
5. Pods schedule onto the new nodes and start draining the queue.
6. As the queue empties, KEDA scales the Deployment back down (eventually to zero, per `minReplicaCount: 0`), and Karpenter's consolidation logic notices the now-underutilized/empty nodes and terminates them.

This is the complete, cost-efficient, demand-driven elasticity story for the interview question: pods scale to match actual event-driven demand (not just CPU, which lags), and nodes scale to match exactly what those pods need (not a fixed pre-provisioned shape), with both scaling back down aggressively when idle.

## Points to Remember

- KEDA scales **pods**, extending HPA with event-source-based scaling (SQS depth, Kafka lag, cron, Prometheus queries, etc.) and the ability to scale to/from zero, which vanilla HPA cannot do.
- KEDA works by managing an HPA object under the hood — it's an addition to HPA (metrics source + scale-to-zero), not a wholesale replacement of it.
- Karpenter scales **nodes**, observing unschedulable pods directly and provisioning exactly-fitting EC2 instances on demand, without needing pre-defined node groups per instance shape.
- Karpenter actively consolidates/bin-packs — it can replace an underutilized node with a better-fitting instance type, not just adjust the count of a fixed type the way Cluster Autoscaler does.
- The two work together in sequence: KEDA scales pod count based on external events → unschedulable pods trigger Karpenter → Karpenter provisions matching nodes → both scale back down when demand drops.

## Common Mistakes

- Using vanilla HPA (CPU/memory only) for a queue-consuming worker, meaning scaling only reacts after CPU usage climbs — a lagging signal compared to queue depth, causing slower response to bursts and unnecessary backlog buildup.
- Assuming HPA can scale to zero replicas — it structurally cannot; that capability specifically requires KEDA (or a similar scale-to-zero controller) layered on top.
- Continuing to run Cluster Autoscaler and Karpenter simultaneously against the same node groups without clearly separating their scope — they can conflict over the same nodes; most teams migrating to Karpenter fully replace Cluster Autoscaler rather than run both against the same capacity.
- Not setting `consolidationPolicy` (or setting it too conservatively) on a Karpenter NodePool, leaving underutilized/expensive nodes running long after the workloads that justified them have scaled down.
- Forgetting that Karpenter needs IAM permissions (via IRSA, same pattern as ESO and the Load Balancer Controller) scoped to actually launch/terminate EC2 instances and modify relevant tags — an under-permissioned Karpenter controller fails silently to provision nodes, and the failure mode looks identical to "pods stuck Pending" caused by other issues, so checking Karpenter's own controller logs is the necessary first diagnostic step.
