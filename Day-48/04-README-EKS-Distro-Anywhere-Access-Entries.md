# Day 48 — EKS Deep Dive: EKS Distro/Anywhere & Access Entries (New RBAC Model)

**Phase:** 1 – Core DevOps | **Week:** W7 | **Domain:** AWS | **Flag:** ⚡ Interview-critical

## Brief

This final file covers two loosely related but both practically important pieces: running EKS-flavored Kubernetes outside AWS's managed control plane entirely (EKS Distro/Anywhere), and the modern replacement for the notoriously fragile `aws-auth` ConfigMap-based cluster authentication model (Access Entries). The latter especially has become a real, current interview topic because it changed a workflow every EKS operator has hit friction with at some point.

## EKS Distro (EKS-D) and EKS Anywhere (EKS-A)

**EKS Distro** is AWS's **open-source Kubernetes distribution** — the exact same versions of Kubernetes, etcd, CoreDNS, and other core components that AWS runs internally for the managed EKS control plane, packaged and released independently so anyone can run the identical, AWS-tested Kubernetes build **without using AWS at all**. It's a set of builds/binaries and container images, not a managed service.

**EKS Anywhere** is the deployment tooling built on top of EKS Distro — a CLI (`eksctl anywhere`) that provisions and lifecycle-manages a full Kubernetes cluster using EKS-D components **on your own infrastructure**: bare metal, VMware vSphere, Docker (for local dev), Snow (AWS Snowball Edge devices), or CloudStack. It brings EKS-like cluster creation/upgrade UX and optional AWS support contracts to environments that can't or won't run in AWS's cloud.

**Why this exists — the real use case**: organizations with strict data-residency or air-gapped/on-prem requirements (regulated industries, edge/retail locations, disconnected environments) who still want Kubernetes version consistency and operational parity with their AWS-hosted EKS clusters, plus an optional paid AWS support relationship for on-prem Kubernetes — without literally running in an AWS region. This matters for a consistent multi-environment strategy: the same Kubernetes version/build runs whether a workload's cluster happens to be an AWS-managed EKS cluster or an on-prem EKS Anywhere cluster, reducing "works on our cloud cluster, breaks on our on-prem cluster" version-skew surprises.

**What's explicitly NOT the case**: EKS-A/EKS-D do not give you the managed EKS control plane (no AWS-operated control plane HA/patching) — you (or your team) are responsible for operating the control plane's HA and upgrades yourself, EKS-A just gives you AWS-built, tested components and lifecycle tooling to do so, plus an optional support contract.

## EKS Access Entries — the new cluster authentication/RBAC model

**The old model (`aws-auth` ConfigMap)**: for years, granting an IAM principal (user/role) access to an EKS cluster meant manually editing a ConfigMap (`aws-auth` in `kube-system`) with a YAML mapping of IAM ARNs to Kubernetes usernames/groups:
```yaml
mapRoles: |
  - rolearn: arn:aws:iam::123456789012:role/admin-role
    username: admin
    groups: ["system:masters"]
```
**Why this was painful in practice**: it's a raw ConfigMap edit with no schema validation, no audit trail beyond whatever tracked the ConfigMap change itself (easy to edit by hand and forget to also update in Terraform, causing drift), a real risk of a malformed YAML edit locking out cluster access entirely, and — critically — a chicken-and-egg bootstrapping problem: only whoever created the cluster (or was already granted access via `aws-auth`) could even edit the ConfigMap to grant access to anyone else, and if that ConfigMap got corrupted, recovering access required AWS support intervention in the worst case.

**The new model — Access Entries**: a first-class EKS API (not a Kubernetes object at all — managed entirely through the AWS API/CLI/Console/Terraform, external to the cluster) mapping an IAM principal directly to Kubernetes RBAC permissions:
```bash
aws eks create-access-entry --cluster-name prod --principal-arn arn:aws:iam::123456789012:role/admin-role
aws eks associate-access-policy --cluster-name prod \
  --principal-arn arn:aws:iam::123456789012:role/admin-role \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster
```

**Why this is a genuine improvement, mechanically:**
- **Managed entirely via the AWS API** — meaning it's fully expressible in Terraform (`aws_eks_access_entry`, `aws_eks_access_policy_association` resources) with real state tracking, plan/diff visibility, and no risk of a hand-edited YAML mistake taking down cluster access.
- **AWS-managed access policies** (`AmazonEKSClusterAdminPolicy`, `AmazonEKSAdminPolicy`, `AmazonEKSEditPolicy`, `AmazonEKSViewPolicy`) provide sensible, versioned RBAC bundles out of the box, scoped either cluster-wide or to a specific namespace (`--access-scope type=namespace,namespaces=team-a`) — no need to hand-write a ClusterRoleBinding for common cases.
- **No chicken-and-egg bootstrapping problem** — because Access Entries are an IAM-permission-gated AWS API operation (`eks:CreateAccessEntry` etc.), not a Kubernetes-object edit, granting the *first* administrator access to a brand-new cluster doesn't require already having Kubernetes-level access — the cluster creator can be automatically granted admin via `enable_cluster_creator_admin_permissions` at creation time (seen in Day 47's Terraform module block) without ever touching `aws-auth`.
- **Coexistence and migration**: clusters can run in `API_AND_CONFIG_MAP` authentication mode during migration (both mechanisms active simultaneously) before cutting over fully to `API` mode — letting teams migrate existing `aws-auth` mappings to Access Entries incrementally rather than a risky big-bang cutover.

## Points to Remember

- EKS Distro is AWS's open-source Kubernetes build (same components as managed EKS); EKS Anywhere is the tooling to run that build on your own infrastructure (bare metal, vSphere, Snow, Docker) with an optional AWS support contract — neither gives you AWS's managed control plane.
- The old `aws-auth` ConfigMap approach to cluster access was a manually-edited, unvalidated Kubernetes object with a real risk of lockout and no clean Terraform-native management path.
- Access Entries are a first-class EKS API (external to the cluster, fully Terraform-manageable) mapping IAM principals directly to Kubernetes RBAC, using AWS-managed access policies scoped to cluster or namespace level.
- Access Entries solve the bootstrapping chicken-and-egg problem — the cluster creator can get admin access automatically via the EKS API without ever needing to edit an in-cluster object first.
- Clusters can run in a hybrid `API_AND_CONFIG_MAP` mode to migrate existing `aws-auth` mappings to Access Entries incrementally before fully cutting over to `API` mode.

## Common Mistakes

- Assuming EKS Anywhere gives you a fully AWS-managed control plane the same way EKS does — it gives you the same tested Kubernetes components and lifecycle tooling, but control-plane operation responsibility stays with you (or your support contract), unlike managed EKS in AWS.
- Manually hand-editing the `aws-auth` ConfigMap on a cluster that's already using Access Entries in `API` mode — in `API`-only mode, `aws-auth` is no longer consulted at all, so edits to it silently have zero effect, confusing anyone who doesn't realize the authentication mode has already been switched.
- Granting `AmazonEKSClusterAdminPolicy` broadly via Access Entries when a narrower, namespace-scoped policy (`AmazonEKSEditPolicy` scoped to one namespace) would satisfy the actual need — same least-privilege discipline that should apply to any IAM policy attachment.
- Forgetting to migrate existing `aws-auth`-based access mappings before fully switching a cluster's authentication mode to `API`-only, unintentionally revoking access for principals that were only ever configured in the old ConfigMap.
- Treating EKS-D/EKS-A as "just Kubernetes you could get anywhere" without acknowledging the actual reason teams choose it — version/build parity with AWS-hosted EKS and an optional AWS support relationship for on-prem/air-gapped environments, not a generic "run Kubernetes somewhere else" recommendation for teams with no such constraint.
