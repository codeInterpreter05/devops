# Day 34 — AWS Compute: EKS, ECS, Lambda: EKS Compute Options

**Phase:** 1 – Core DevOps | **Week:** W5 | **Domain:** AWS | **Flag:** —

## Brief

If you've spent any time with self-managed Kubernetes (kubeadm, kOps), EKS is the same Kubernetes API surface with AWS operating (some of) the parts that hurt most — the control plane. But "run Kubernetes on EKS" still leaves a real decision: how do the *worker nodes* actually run — self-managed EC2 you patch yourself, managed node groups AWS partially automates, or Fargate where you never see a node at all. This is a genuinely operational decision with real cost/control tradeoffs, not just a checkbox, and interviewers probe it to see if you understand what you're trading away at each level.

This day is split into three files:

1. **This file** — EKS's compute options: self-managed nodes, managed node groups, and Fargate.
2. **[02-README-ECS-vs-EKS.md](02-README-ECS-vs-EKS.md)** — when to choose ECS over EKS, and the operational tradeoffs.
3. **[03-README-Lambda-And-EventBridge.md](03-README-Lambda-And-EventBridge.md)** — Lambda cold starts, provisioned concurrency, layers, and EventBridge.

## What EKS actually manages for you

EKS runs and operates the Kubernetes **control plane** (API server, etcd, scheduler, controller manager) across multiple AZs, patches it, and handles its scaling and upgrades — you never SSH into a control plane node, and you don't manage etcd backups yourself. What EKS does **not** decide for you is how your **worker nodes** — the machines that actually run your pods — are provisioned and managed. That's a separate, explicit choice with three tiers of increasing abstraction.

## Self-managed nodes — full control, full operational burden

```bash
eksctl create nodegroup --cluster my-cluster --managed=false --node-type t3.medium --nodes 3
```

You provision EC2 instances yourself (via an Auto Scaling Group, launch template, or `eksctl`), install the correct kubelet/container runtime version, join them to the cluster, and are personally responsible for OS patching, kubelet upgrades, and coordinating node AMI updates with your Kubernetes version. This is the most work but also the most control — needed when you require a custom AMI, a specific kernel module, non-standard instance bootstrapping, or GPU driver configurations that managed node groups don't support out of the box.

## Managed node groups — AWS automates the EC2 lifecycle

```bash
eksctl create nodegroup --cluster my-cluster --managed --node-type t3.medium --nodes 3 --nodes-min 2 --nodes-max 6
```

AWS provisions and manages the underlying Auto Scaling Group for you, using **EKS-optimized AMIs** it maintains and patches. Managed node groups add real operational conveniences self-managed nodes lack:
- **Managed node updates/upgrades**: `eksctl upgrade nodegroup` or a console click rolls nodes through a controlled drain-and-replace cycle, respecting Pod Disruption Budgets — you don't hand-write the drain/cordon/replace logic yourself.
- **Automatic AMI patching workflow** — new EKS-optimized AMIs (with security patches) get surfaced, and updating the node group rolls them out safely.
- Still **EC2 under the hood** — you still pay per-instance, still choose instance types/sizes, and workloads still share nodes (multi-tenancy at the node level, same `kubelet`/resource-request considerations as any Kubernetes cluster).

This is the default, sensible choice for most teams: real cost efficiency of standard EC2 (including Spot support for cost savings), most of the patching/lifecycle toil automated away, while keeping full control over instance types, node taints, and custom `kubelet` configuration via launch templates.

## Fargate — no nodes at all

```bash
eksctl create fargateprofile --cluster my-cluster --namespace my-app
```

With **EKS Fargate**, there is no visible EC2 node — AWS runs each pod in its own **isolated micro-VM**, sized automatically to the pod's resource requests, with no shared kernel between pods (a materially stronger isolation boundary than containers sharing a node's kernel — relevant for genuinely multi-tenant workloads). You define a **Fargate profile** matching namespaces/labels, and any pod matching it gets scheduled onto Fargate instead of a regular node.

Tradeoffs to know cold:
- **No DaemonSets** — since there's no persistent node to run one on. Node-level agents (log shippers, service mesh sidecars implemented as DaemonSets) need a different pattern (sidecar containers injected per-pod instead).
- **Slower pod startup** — provisioning a fresh micro-VM per pod is slower than scheduling onto an already-running node (seconds, not the sub-second scheduling of a warm node with capacity).
- **Pricing is per-pod, by vCPU/memory requested**, rounded up to fixed increments — cost-predictable for spiky, low-density workloads, but often *more expensive* than a well-bin-packed managed node group at steady, high-density utilization, since Fargate can't share a machine's spare capacity across many small pods the way node-level bin-packing does.
- **No control over the underlying instance** — can't tune kernel parameters, can't run privileged DaemonSets, can't choose GPU instance types (as of general availability, GPU support on Fargate is far more limited than on managed/self-managed nodes).

Fargate earns its place for workloads valuing strong per-pod isolation or operational simplicity over cost efficiency and node-level customization — batch jobs, low/spiky traffic services, or multi-tenant SaaS platforms isolating customer workloads from each other at the VM level, not just the container level.

## Points to Remember

- EKS always manages the control plane; the worker-node tier is a separate, explicit choice with three levels of abstraction (self-managed, managed node groups, Fargate).
- Managed node groups automate the EC2 lifecycle (patching, controlled node replacement respecting PDBs) while you still choose instance types and still pay per-instance — the sensible default for most teams.
- Fargate removes the node entirely — one micro-VM per pod, no DaemonSets, slower cold-start scheduling, and pricing per-pod that can be more expensive than well-utilized nodes at scale.
- Self-managed nodes are the highest-effort, highest-control tier — needed for custom AMIs/kernels/bootstrapping that managed node groups don't support.
- The isolation model differs meaningfully: Fargate's per-pod micro-VM is a stronger boundary than containers sharing a node's kernel, which matters for genuine multi-tenant security requirements.

## Common Mistakes

- Choosing Fargate by default for cost savings without modeling actual bin-packing efficiency — a densely-packed managed node group is very often cheaper at steady-state than per-pod Fargate billing.
- Deploying a DaemonSet-based tool (log agent, node exporter) into a Fargate-only cluster and being surprised it never schedules — DaemonSets require an actual persistent node.
- Treating managed node groups as fully hands-off — you still own choosing/right-sizing instance types, setting up cluster autoscaling (or Karpenter), and deciding on Spot vs. On-Demand mix; AWS automates the *lifecycle* operations, not capacity planning.
- Assuming self-managed nodes are never worth the extra effort — for GPU workloads with specific driver/kernel requirements, or heavily customized node bootstrapping, they're sometimes still the only option that actually works.
- Forgetting that Fargate pod startup latency (provisioning a fresh micro-VM) is materially slower than scheduling onto warm node capacity — a poor fit for workloads needing very fast, bursty autoscaling response times.
