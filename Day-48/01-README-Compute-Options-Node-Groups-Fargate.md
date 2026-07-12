# Day 48 — EKS Deep Dive: Managed Node Groups vs. Fargate

**Phase:** 1 – Core DevOps | **Week:** W7 | **Domain:** AWS | **Flag:** ⚡ Interview-critical

## Brief

By this point you've built EKS clusters (Day 47) and scheduled workloads onto them (Day 45) — this day steps back to the compute layer question every EKS cluster design starts with: how do pods actually get a machine to run on? The answer shapes everything downstream — networking model, autoscaling behavior, cost, and operational burden. This is also flagged interview-critical because "explain the EKS networking model, how does VPC CNI differ from other CNI plugins" (this day's interview question) only makes sense once you understand how compute is provisioned in the first place.

This day is split into four files:

1. **This file** — Managed Node Groups vs. Fargate, the two fundamental compute models.
2. **[02-README-EKS-Networking-Addons.md](02-README-EKS-Networking-Addons.md)** — EKS add-ons (CoreDNS, kube-proxy, VPC CNI) and the AWS Load Balancer Controller.
3. **[03-README-Karpenter-vs-Cluster-Autoscaler.md](03-README-Karpenter-vs-Cluster-Autoscaler.md)** — the two node-autoscaling approaches.
4. **[04-README-EKS-Distro-Anywhere-Access-Entries.md](04-README-EKS-Distro-Anywhere-Access-Entries.md)** — EKS Distro/Anywhere and the new access entries RBAC model.

## Self-managed nodes vs. Managed Node Groups

Before comparing to Fargate, it's worth being precise about the EC2-based option itself. **Self-managed node groups** are plain EC2 Auto Scaling Groups you configure and patch yourself (via a custom launch template, your own AMI choice, your own bootstrap script calling `/etc/eks/bootstrap.sh`) — full control, full operational burden. **Managed Node Groups (MNG)** are AWS-orchestrated EC2 Auto Scaling Groups where AWS handles provisioning, the EKS-optimized AMI selection/updates, graceful node draining during updates/termination, and lifecycle integration with the EKS control plane (e.g., automatically labeling/tainting nodes, handling `SIGTERM`-based cordoning during a managed update) — you still pick instance types, sizes, and scaling bounds, but AWS owns the underlying automation. MNG is the default recommended choice for most EC2-based EKS compute today; self-managed node groups exist mainly for cases needing tighter AMI/bootstrap customization than MNG allows.

## Fargate for EKS

**Fargate** removes node management entirely — you define a **Fargate Profile** matching namespace + (optionally) label selectors, and any pod matching that profile gets its own dedicated, right-sized micro-VM with no visible underlying EC2 instance to patch, size, or manage.

```yaml
# Fargate profile (created via eksctl/Terraform, not a Kubernetes object)
# selectors: namespace: batch-jobs, labels: {compute: fargate}
```

**What you give up and gain, precisely:**
- **No node-level DaemonSets** — Fargate pods can't use DaemonSets (no shared node to run them on), which rules out node-level log/metrics agents that rely on a DaemonSet pattern (e.g., a traditional Fluent Bit or Datadog node-agent DaemonSet) — Fargate has its own separate logging integration path (a `aws-logging` ConfigMap piping directly to CloudWatch) instead.
- **No `hostNetwork`/`hostPort`/privileged pods** — each Fargate pod is isolated in its own micro-VM boundary, so anything requiring host-level access is unsupported.
- **No GPU support** — Fargate for EKS doesn't support GPU-backed pods; anything ML/GPU-workload related must run on managed/self-managed EC2 node groups.
- **Billing is per-pod, per-vCPU/memory-second** — not per-node — meaning you pay exactly for what each pod requests (rounded up to Fargate's supported size increments), with no idle node capacity to pay for, but generally a higher per-unit compute cost than equivalent EC2 pricing (before Spot/Savings Plans).
- **Startup latency** — a Fargate pod's cold start (allocating and booting a new micro-VM) is typically slower than scheduling onto an already-running EC2 node with spare capacity — matters for latency-sensitive autoscaling scenarios.

## Decision framework — this is the interview-relevant part

- **Fargate fits**: bursty/unpredictable batch or CI-style workloads where you don't want to manage node capacity planning at all, workloads with strict per-pod security isolation requirements (no shared kernel/node with other tenants' pods), or a small platform team that explicitly wants to trade some cost/flexibility for zero node-patching operational burden.
- **Managed Node Groups fit**: anything needing DaemonSets (observability agents, service mesh sidecarless proxies, node-level security agents), GPU workloads, workloads sensitive to Fargate's relative cost premium at scale, or anything needing fine-grained node-level tuning (custom AMI, specific kernel parameters, instance-store NVMe access).
- **Most real production EKS clusters run both**: Managed Node Groups (increasingly provisioned dynamically via Karpenter rather than static MNGs — see file 03) as the default compute layer for most workloads, with a Fargate Profile carved out for specific namespaces (e.g., a `kube-system` add-on that must never be affected by node-level issues, or a CI/batch namespace that benefits from Fargate's pay-per-pod isolation) — not an either/or choice at the cluster level, a per-namespace/per-workload one.

## Points to Remember

- Managed Node Groups automate EC2 lifecycle (AMI selection, graceful draining, scaling) but you still choose instance types/sizes; self-managed node groups give full control at full operational cost.
- Fargate for EKS gives per-pod isolated micro-VMs with zero node management, at the cost of no DaemonSets, no `hostNetwork`/privileged pods, no GPU support, and typically higher relative compute cost plus slower cold-start latency.
- Fargate pods need a dedicated logging path (the `aws-logging` ConfigMap to CloudWatch) since the DaemonSet-based log-agent pattern isn't available to them.
- The real-world decision is usually per-namespace/per-workload, not cluster-wide — many clusters mix Managed Node Groups (or Karpenter-provisioned nodes) with Fargate Profiles for specific isolated workloads.
- GPU workloads and anything needing DaemonSets structurally require EC2-based compute — Fargate is not an option for those regardless of other tradeoffs.

## Common Mistakes

- Assuming Fargate is strictly "cheaper because you don't pay for idle nodes" without accounting for its higher per-unit compute pricing relative to well-utilized, right-sized EC2 — Fargate's savings depend heavily on how poorly utilized your EC2 nodes would otherwise be.
- Trying to deploy a DaemonSet-based observability/security agent onto a Fargate Profile's namespace and being confused when it never schedules — DaemonSets are structurally incompatible with Fargate.
- Choosing Fargate for a latency-sensitive, frequently-scaling workload without accounting for cold-start latency being meaningfully slower than scheduling onto pre-warmed EC2 capacity.
- Treating "Managed Node Groups" and "self-managed node groups" as synonyms — they have materially different operational burden (AWS-orchestrated lifecycle/updates vs. fully your own responsibility), and this distinction is a real interview trap.
- Forgetting GPU workloads cannot run on Fargate at all, and only discovering this after already committing a cluster design assuming full Fargate coverage.
