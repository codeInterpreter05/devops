# Day 121-130 — Multi-Cloud & Edge I: GKE Deep Dive vs. EKS

**Phase:** 4 – Advanced/Specialization | **Week:** W21-W22 | **Domain:** Advanced | **Flag:** —

## Brief

This is a 10-day technical-deep-dive block, not a review block — new content, going deep on real mechanisms. The core skill being built: you should be able to stand up a working cluster on two different clouds, explain *precisely* where their architectures diverge (not just "GKE is Google's, EKS is Amazon's"), and reason concretely about the cost and networking consequences of that divergence. This matters for real multi-cloud/avoid-lock-in conversations, which show up constantly in senior DevOps/platform interviews.

This block is split into four files:

1. **This file** — GKE's architecture end-to-end and how it compares to EKS: control plane management, node pools, Autopilot vs. Standard, and the networking model.
2. **[02-README-AKS-Anthos-Azure-Arc.md](02-README-AKS-Anthos-Azure-Arc.md)** — AKS specifics, then Anthos and Azure Arc as the two competing approaches to managing fleets of clusters across clouds.
3. **[03-README-ClusterAPI-Multicluster-Istio.md](03-README-ClusterAPI-Multicluster-Istio.md)** — Cluster API for declarative cluster lifecycle/federation across infrastructure providers, and multi-cluster Istio for cross-cluster service mesh.
4. **[04-README-Cloudflare-Workers-Egress-Economics.md](04-README-Cloudflare-Workers-Egress-Economics.md)** — Cloudflare Workers as an edge-compute model, and the real, quotable economics of cross-cloud data egress.

The hands-on build-out (standing up GKE and EKS, then CAPI, then multi-cluster Istio, then a cost teardown) lives in [LAB.md](LAB.md) — read the READMEs first for the *why*, then go build it there.

## GKE's control plane: what "fully managed" actually means

On GKE, you never see, provision, patch, or pay a line-item for master/control-plane nodes in **Standard** mode — Google runs the API server, etcd, scheduler, and controller-manager in Google-owned infrastructure, entirely outside your project's compute billing (a small **cluster management fee** applies per cluster beyond one free zonal cluster per billing account, separate from node cost). Concretely:

- **Regional vs. zonal clusters.** A **zonal** cluster has a single control plane replica in one zone — a zone outage takes the control plane down (existing pods keep running, but you can't schedule/`kubectl apply` anything until it's back). A **regional** cluster runs the control plane across three zones with automatic leader election and failover — this is the production default, and it's what backs GKE's 99.95% SLA.
- **Upgrades are automatic** by default, governed by a **release channel**: `RAPID` (newest, least soak time), `REGULAR` (default, several weeks of soak), `STABLE` (longest soak, most conservative). You can pin a maintenance window/exclusion but you don't hand-run `kubeadm upgrade` — Google does it, and node pools auto-upgrade to track the control plane (within GKE's supported skew).
- **You cannot SSH into or inspect the control plane** — no `kubectl exec` into `kube-apiserver`, no reading its logs directly (Cloud Logging/Cloud Audit Logs are your only visibility). This is the tradeoff for "fully managed": less operational burden, less low-level debugging surface.

```bash
# Standard, VPC-native, REGIONAL cluster — the production-shaped default
gcloud container clusters create multicloud-demo \
  --region=us-central1 \
  --release-channel=regular \
  --num-nodes=1 \
  --machine-type=e2-medium \
  --enable-ip-alias \
  --workload-pool=$(gcloud config get-value project).svc.id.goog
```

## EKS's control plane: managed, but you still see the seams

EKS's control plane is also run by AWS (multi-AZ by default, HA out of the box) — but the abstraction is thinner in three concrete ways that come up constantly in comparisons:

- **You pay a per-cluster hourly fee** ($0.10/hr as of this writing, ~$73/month per cluster) regardless of node count — a line item GKE Standard doesn't have in the same form (GKE's fee structure differs and has a free-cluster allowance).
- **Control plane and node upgrades are two separate, manual operations.** `eksctl upgrade cluster` (or the console/API) bumps the control plane's Kubernetes version; node groups then need their own explicit upgrade (`eksctl upgrade nodegroup`) and must stay within one minor version of the control plane, or the cluster is in an unsupported/degraded state. Nothing auto-upgrades unless you build automation around it.
- **Fargate profiles** give you a pod-level "no nodes to manage" option (closer to Autopilot's philosophy) for specific namespaces/labels, while everything else on the cluster can still run on EC2-backed managed node groups — a mixed model, whereas GKE Autopilot is an all-or-nothing cluster-wide mode.

```bash
eksctl create cluster \
  --name multicloud-demo \
  --region us-east-1 \
  --version 1.29 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 2 \
  --managed
```

## Autopilot vs. Standard — GKE's real differentiator

- **Standard**: you choose machine types, manage node pools, can run DaemonSets, hostPath volumes, privileged containers, custom kubelet flags — full control, full responsibility (you patch nodes, size them, handle bin-packing).
- **Autopilot**: Google provisions and scales nodes for you; you request pod-level CPU/memory and Google fits them onto right-sized infrastructure it manages. Billing is **per-pod resource request**, not per-node — so an idle over-provisioned node pool (a classic Standard-mode cost leak) structurally can't happen. In exchange, Autopilot **blocks** a set of things by policy: no privileged pods, no hostPath/hostNetwork/hostPort, no DaemonSets in most cases (system DaemonSets are Google-managed), and it enforces baseline Pod Security Standards automatically.

```bash
gcloud container clusters create-auto multicloud-demo-autopilot --region=us-central1
```

EKS's closest equivalent isn't a single flag — it's **Fargate profiles** layered onto an otherwise normal cluster:

```bash
eksctl create fargateprofile \
  --cluster multicloud-demo \
  --name fp-default \
  --namespace default
```

The practical difference to say out loud in an interview: **Autopilot is a cluster-wide operating mode**; **EKS Fargate is a per-namespace/per-label escape from node management**, coexisting with regular managed node groups on the same cluster.

## Networking model — where the real architectural divergence is

This is the section that separates "I've used both" from "I understand why they behave differently under load."

**GKE:**
- **VPC-native (alias IP) clusters** are the default and near-universal choice today — pods get IPs from a secondary CIDR range on the node's VPC subnet (`--enable-ip-alias`), which means pod IPs are natively routable inside the VPC without any overlay or NAT — Cloud Load Balancers, firewall rules, and VPC peering all see real pod IPs directly.
- **GKE Dataplane V2** (Cilium-based eBPF, the modern default) replaces kube-proxy's iptables rule chains with eBPF programs for service routing and enforces `NetworkPolicy` **natively** — no separate Calico/Cilium install needed for policy enforcement, and it scales far better than iptables at high Service/endpoint counts.
- Egress from pods to the internet goes through **Cloud NAT** (if nodes don't have public IPs, the common production pattern) — this is billed per-GB processed, a real line item at scale (see file 4).

**EKS:**
- The default **VPC CNI plugin** assigns each pod a **real ENI-backed IP** from the VPC subnet too — but ENIs have a hard per-instance-type limit (a `t3.medium` supports far fewer ENI-backed pod IPs than a larger instance), which is a **common EKS-specific capacity surprise**: you can run out of pod IPs on small instance types well before you run out of CPU/memory. **Prefix delegation** (`ENABLE_PREFIX_DELEGATION=true`) mitigates this by assigning /28 prefixes instead of individual IPs per ENI slot, multiplying effective pod density per node.
- kube-proxy defaults to iptables mode on EKS; **Cilium can be installed as a CNI replacement** for eBPF-based dataplane behavior (closer to GKE Dataplane V2's model), but it's an explicit, separate choice — not the out-of-the-box default the way it now is on GKE.
- Egress similarly requires a **NAT Gateway** per AZ for private-subnet nodes — billed both per-hour and per-GB processed (a frequently underestimated EKS cost line, covered concretely in file 4).

## Identity and RBAC — Workload Identity vs. IRSA

Both clouds solved the same underlying problem (a pod needs to call a cloud API without embedding long-lived credentials) with a structurally similar pattern:

- **GKE Workload Identity**: binds a Kubernetes ServiceAccount to a Google Cloud IAM ServiceAccount via a trust relationship on the GCP IAM side (`--workload-pool` at cluster creation, then an IAM policy binding). The pod's token exchange happens transparently through GKE's metadata server proxy — no static key files, ever.
- **EKS IRSA (IAM Roles for Service Accounts)**: associates an **OIDC provider** with the cluster, then annotates a Kubernetes ServiceAccount with an IAM role ARN. The pod gets short-lived credentials via a projected service account token exchanged for AWS STS credentials.

```bash
# EKS IRSA setup
eksctl utils associate-iam-oidc-provider --cluster multicloud-demo --approve
eksctl create iamserviceaccount \
  --cluster multicloud-demo --namespace default --name hello-eks-sa \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve
```

Both are the correct modern pattern (never mount static cloud credentials into a pod); the concrete difference worth naming in an interview is **where the trust boundary lives** — GCP's IAM policy binding vs. AWS's OIDC federation + IAM role trust policy — but the *outcome* (short-lived, pod-scoped, no static keys) is functionally equivalent.

## Points to Remember

- GKE Standard's control plane is invisible and mostly hands-off (auto-upgrade via release channels); EKS's control plane is managed but you still explicitly drive control-plane and node-group upgrades as separate steps and pay a per-cluster hourly fee.
- Autopilot is a cluster-wide billing/operating mode (per-pod billing, policy-restricted); EKS Fargate is a per-namespace/label opt-out from node management layered onto an otherwise normal node-group cluster — not the same shape of tradeoff.
- Both clouds' default CNI gives pods real, VPC-routable IPs (no overlay) — but EKS's ENI-based IP allocation has a real per-instance-type pod-density ceiling that GKE's alias-IP model doesn't hit the same way; prefix delegation is the EKS-specific fix.
- GKE ships eBPF-based Dataplane V2 (Cilium) as the default dataplane with native NetworkPolicy enforcement; EKS defaults to iptables kube-proxy and requires an explicit Cilium install to get the same eBPF model.
- Workload Identity (GKE) and IRSA (EKS) solve the same problem — pod-scoped, short-lived cloud credentials with no static keys — via different trust-federation mechanics (IAM policy binding vs. OIDC + IAM role trust policy).

## Common Mistakes

- Assuming EKS node upgrades happen automatically like GKE's release channels — they don't; an EKS cluster left un-upgraded for too long can drift more than one minor version behind and land in an unsupported state that blocks a smooth upgrade path.
- Sizing EKS node instance types purely on CPU/memory and then hitting a "pod IP" scheduling wall caused by ENI limits — always check max-pods-per-node for the chosen instance type, not just resource requests.
- Treating Autopilot and Fargate as directly equivalent in scope — Autopilot changes the whole cluster's operating model; Fargate profiles are scoped to specific namespaces/pod selectors on an otherwise conventional EKS cluster.
- Forgetting that GKE's regional (multi-zone control plane) vs. zonal distinction is a real availability decision, not a cosmetic one — a zonal cluster's control plane is a single point of failure for control-plane operations (not for already-running workloads).
- Mounting a static IAM/service-account key into a pod instead of using Workload Identity/IRSA "because it was faster to set up" — this is a durable, high-blast-radius credential the moment it leaks, versus a short-lived token that expires.
