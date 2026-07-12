# Day 51 — K8s Upgrades & Cluster Management: EKS Upgrade Strategy

**Phase:** 1 – Core DevOps | **Week:** W8 | **Domain:** Kubernetes | **Flag:** —

## Brief

Kubernetes ships a new minor version roughly every four months, and each minor version is only supported (by upstream and by managed offerings like EKS) for a limited window before it goes end-of-life. That means "upgrading the cluster" isn't a one-time project — it's a recurring operational responsibility, and doing it wrong causes exactly the kind of production incident that ends up in a postmortem. This is the topic behind today's interview question: "how do you upgrade an EKS cluster with zero downtime and minimal risk?" Understanding the order of operations and why that order is mandatory is what separates a real answer from a guess.

This day is split into three files:

1. **This file** — EKS upgrade strategy: control plane first, node upgrade approaches, and blue/green cluster upgrades.
2. **[02-README-Workload-Safety-During-Upgrades.md](02-README-Workload-Safety-During-Upgrades.md)** — PodDisruptionBudgets, drain, and cordon.
3. **[03-README-Deprecation-And-Backup.md](03-README-Deprecation-And-Backup.md)** — `kubectl-convert`/`pluto` for API deprecations, and Velero for cluster backup.

## Why control plane must be upgraded before nodes

Kubernetes has a strict **version skew policy**: the control plane (API server) must always be at a version **greater than or equal to** every kubelet's version, and kubelets can be at most **three minor versions behind** the API server (the exact skew window has varied across Kubernetes releases — historically 2, now widened to 3 for kubelet-to-control-plane in recent versions; the direction of the rule is what matters). Nodes can *never* run a newer Kubernetes version than the control plane.

This is why the mandatory order is: **upgrade the control plane first, then upgrade nodes**. If you upgraded nodes first, you could end up with kubelets speaking a newer API dialect than the control plane understands — the API server might reject fields/behaviors the newer kubelet expects, or the newer kubelet might use APIs the older control plane hasn't implemented yet. EKS enforces this directly: you cannot upgrade a node group to a Kubernetes minor version newer than the current control plane version — the API/console will simply refuse.

**EKS-specific mechanics:**
```bash
# 1. Upgrade control plane (one minor version at a time — cannot skip versions)
aws eks update-cluster-version --name my-cluster --kubernetes-version 1.29

# 2. Wait for control plane upgrade to complete
aws eks describe-update --name my-cluster --update-id <UPDATE_ID>

# 3. Only now upgrade node groups
aws eks update-nodegroup-version --cluster-name my-cluster --nodegroup-name my-nodegroup
```
EKS control plane upgrades are **one minor version at a time** — you cannot jump from 1.27 straight to 1.29; you must pass through 1.28. This is a deliberate safety constraint since each minor version can carry breaking API changes, and skipping versions means skipping the chance to catch a breaking change before it compounds with another.

Also update **add-ons** (VPC CNI, CoreDNS, kube-proxy, EBS/EFS CSI drivers) to versions compatible with the new control plane version — an outdated CNI plugin after a control plane bump is a common source of "cluster upgraded fine but new pods can't get an IP" incidents.

```bash
aws eks describe-addon-versions --addon-name vpc-cni --kubernetes-version 1.29
aws eks update-addon --cluster-name my-cluster --addon-name vpc-cni --addon-version <VERSION>
```

## Node upgrade approaches

Once the control plane is upgraded, nodes need to move to the new version too (an EKS control plane will keep serving older nodes for a while, but only within the supported skew — you can't leave nodes on an old version indefinitely).

- **In-place / rolling node group upgrade**: EKS-managed node groups support `update-nodegroup-version`, which performs a **rolling replacement** — cordons and drains nodes respecting any PodDisruptionBudgets, launches new nodes on the new AMI/version, and terminates old ones, a configurable batch at a time. Simplest to operate, uses the existing node group, but you're modifying live infrastructure and any misbehavior on the new AMI shows up mid-migration.
- **Blue/green node group (safer, more resource-intensive)**: create an entirely **new** node group already running the target Kubernetes/AMI version, alongside the existing one. Cordon the old node group (stop new scheduling there), let workloads gradually reschedule/drain onto the new node group, verify health, then scale the old node group to zero and delete it. This avoids ever running a partially-upgraded rolling batch in the critical path — if the new node group has a problem, the old one is still fully intact and you simply don't cut over, rather than having to roll back a partially-completed rolling update.
- **Blue/green at the *cluster* level** (most conservative, used for major version jumps or when you don't trust in-place control plane upgrades for a critical cluster): stand up an entirely new EKS cluster at the target version, deploy workloads to it via GitOps (the same manifests/Helm charts you already have), validate with real or shadow traffic, then cut over DNS/load balancer weight gradually (canary-style), and decommission the old cluster only once the new one has proven stable. More expensive (running two clusters simultaneously) and more work (data migration for anything stateful), but it's the only approach with a true, trivial rollback (just shift traffic back) if something is wrong with the new cluster at a level deeper than a single node group — e.g., a control plane behavior change.

**When to choose which**: rolling node group upgrades for routine, well-tested minor version bumps on non-critical clusters; blue/green node groups for anything customer-facing where you want a cheap safety net; full blue/green cluster upgrades for major jumps, first time upgrading past a version with known breaking changes, or clusters where an extended partial-outage during upgrade is unacceptable.

## Points to Remember

- Control plane is always upgraded before nodes — nodes can never run a newer Kubernetes minor version than the control plane, and EKS enforces this at the API level.
- EKS control plane upgrades are one minor version at a time — no skipping versions, even if you're several versions behind.
- Add-ons (VPC CNI, CoreDNS, kube-proxy, CSI drivers) need version compatibility checks alongside the control plane bump — an upgrade isn't done until these are current too.
- Rolling node group upgrade = simplest, modifies existing infrastructure in place. Blue/green node group = safer, runs old and new side-by-side with an easy abort. Blue/green cluster = most conservative, full workload cutover with trivial rollback, at the cost of running two clusters and migrating stateful data.
- The whole point of "control plane first" ordering is the version-skew policy — kubelets can be behind the API server by a bounded number of minor versions, never ahead.

## Common Mistakes

- Attempting to upgrade a node group directly to a Kubernetes version newer than the current control plane — EKS will reject it, but teams sometimes discover this mid-maintenance-window instead of planning the order up front.
- Skipping the add-on version check after a control plane upgrade, leading to subtle failures (new pods stuck in `ContainerCreating` due to an incompatible CNI version) that look unrelated to the upgrade itself.
- Trying to jump multiple EKS minor versions in one upgrade operation — not supported; each skipped version is also a skipped opportunity to catch that version's specific breaking changes in isolation.
- Choosing a rolling in-place node group upgrade for a customer-facing production cluster with tight SLAs, then having no easy way to abort partway through when a new AMI turns out to have a driver or kernel issue — for high-stakes clusters, the extra cost of blue/green is usually worth the trivial rollback.
- Forgetting that stateful workloads (databases, anything with PVs) need an explicit data migration plan for a blue/green *cluster* upgrade — you can't just "point workloads at the new cluster" the way you can for stateless services.
