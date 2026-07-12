# Day 48 — EKS Deep Dive: Karpenter vs. Cluster Autoscaler

**Phase:** 1 – Core DevOps | **Week:** W7 | **Domain:** AWS | **Flag:** ⚡ Interview-critical

## Brief

Node-level autoscaling is where a lot of real EKS cost and reliability problems live — provision too conservatively and pods sit `Pending` during a traffic spike; too generously and you're paying for idle capacity around the clock. Karpenter, a newer AWS-originated tool, has become the modern default over the older Cluster Autoscaler for reasons that go beyond "it's newer" — this is also this day's hands-on activity, replacing one with the other and benchmarking the difference.

## Cluster Autoscaler — the original approach

Cluster Autoscaler (CA) works **at the Auto Scaling Group (ASG) level** — it doesn't talk to EC2 directly; it watches for `Pending` pods that can't be scheduled due to insufficient resources, figures out which of your **pre-defined node groups/ASGs** could accommodate them if scaled up, and calls the ASG's scaling API to add nodes to that specific group.

**The structural limitation this creates**: CA can only scale **existing** node groups up or down — it cannot decide "actually, a different instance type would fit this pod better." You (or your Terraform config) must pre-define node groups for every instance-type/AZ combination you might need, and CA picks among them. This leads to either:
- **Over-provisioning many node groups** "just in case" (one per instance family/size you might need), adding real operational and Terraform-maintenance overhead, or
- **Under-provisioning flexibility**, where a pod with a slightly unusual resource request (e.g., needing more memory than any predefined node group's instance type comfortably fits) sits `Pending` even though a perfectly good instance type exists in EC2 — CA just doesn't know to ask for it because no node group was configured for it.

Scale-down in CA works similarly — it identifies underutilized nodes within existing groups and cordons/drains them, respecting PodDisruptionBudgets, but decision-making is still constrained to "which of my predefined groups has a node I can remove."

## Karpenter — direct-to-EC2, no node groups

Karpenter (originally AWS-built, now a CNCF project) eliminates the ASG/node-group middle layer entirely. It watches for unschedulable pods directly and **calls the EC2 `RunInstances`/`CreateFleet` API directly**, choosing the actual instance type, size, AZ, and capacity type (on-demand vs. spot) that best fits the pending pods' real resource requests — computed fresh at scale-up time, not chosen from a small pre-defined menu.

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata: {name: default}
spec:
  template:
    spec:
      requirements:
      - key: karpenter.sh/capacity-type
        operator: In
        values: ["spot", "on-demand"]
      - key: kubernetes.io/arch
        operator: In
        values: ["amd64", "arm64"]
      nodeClassRef: {name: default}
  limits: {cpu: "1000"}
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 30s
```

**Why this is faster and more efficient, concretely:**
- **Scale-up latency**: CA's ASG-based scaling has an inherent extra hop (CA → ASG desired-count change → ASG API → EC2 `RunInstances`), plus ASG's own scaling-activity overhead; Karpenter calls EC2 directly, typically shaving real seconds-to-low-minutes off node provisioning time — this is exactly what this day's benchmark lab is measuring.
- **Bin-packing / right-sizing**: Karpenter evaluates the *actual* pending pods' resource requests and picks the smallest/cheapest instance type(s) that fit them (constrained by whatever `requirements` you set — architecture, capacity type, zone, instance family), rather than scaling a fixed-shape node group that may over- or under-fit the actual demand.
- **Consolidation**: Karpenter continuously looks for opportunities to replace multiple underutilized nodes with fewer, better-fitting ones (or remove empty nodes) — an ongoing bin-packing optimization CA doesn't do (CA only scales down, it doesn't proactively repack).
- **No pre-defined node groups to maintain**: a single `NodePool` (or a small handful, e.g., separate ones for spot vs. on-demand or GPU vs. CPU workloads) replaces potentially dozens of hand-maintained ASGs/managed node groups in Terraform.

**Real-world caveats worth knowing:**
- Karpenter still respects taints/tolerations, node affinity, and topology spread constraints (Day 45) when choosing where/what to provision — it's a provisioner, not a scheduler override; the *Kubernetes* scheduler still ultimately places pods, Karpenter just makes sure suitable nodes exist.
- Migrating from CA to Karpenter is usually done gradually — running both temporarily with CA's node groups scaled to a floor while Karpenter takes over new capacity, then decommissioning the old managed node groups once confidence is established (exactly the pattern in this day's hands-on lab).
- Karpenter needs its own IAM permissions (via IRSA) to call EC2 APIs directly (`ec2:RunInstances`, `ec2:CreateFleet`, `ec2:TerminateInstances`, etc.) — a materially broader permission set than CA needs (which only touches ASG APIs), worth flagging in a security review.

## Points to Remember

- Cluster Autoscaler scales existing, pre-defined node groups/ASGs up or down — it cannot choose a different instance type than what a matching node group already defines.
- Karpenter calls EC2 APIs directly, computing the best-fit instance type/size/AZ/capacity-type per batch of pending pods at scale-up time — no pre-defined node groups required.
- Karpenter's consolidation feature proactively repacks/removes underutilized nodes on an ongoing basis; CA only reactively scales down, it doesn't repack.
- Karpenter needs broader IAM permissions (direct EC2 instance lifecycle APIs) than CA (ASG-scoped APIs only) — a real security-review talking point.
- Both respect Kubernetes-level scheduling constraints (taints, affinity, topology spread) — they provision capacity, they don't override where the scheduler ultimately places pods.

## Common Mistakes

- Assuming Cluster Autoscaler can "pick the best instance type" for a pending pod — it can only choose among pre-defined node groups; an unusual pod resource shape with no matching node group stays `Pending` even if a good EC2 instance type exists.
- Migrating to Karpenter without granting it the broader EC2-level IAM permissions it needs (versus CA's narrower ASG-only permissions), then being confused why it can't provision anything.
- Running Karpenter with an overly permissive `NodePool` (no `limits`, broad instance-type requirements) and being surprised by unexpectedly large or expensive instance types getting provisioned for oddly-shaped pod requests.
- Not accounting for Karpenter's consolidation behavior potentially causing more node churn (nodes being replaced to optimize packing) than teams used to CA's more conservative scale-down-only behavior expect — worth tuning `consolidationPolicy`/`consolidateAfter` deliberately rather than leaving defaults unexamined for disruption-sensitive workloads.
- Forgetting that Karpenter still needs to interoperate with taints/affinity/topology spread rules from Day 45 — a `NodePool` that doesn't account for your spot-node taint convention, for instance, can end up conflicting with or duplicating existing scheduling policy.
