# Day 121-130 — Multi-Cloud & Edge II: AKS, Anthos, and Azure Arc

**Phase:** 4 – Advanced/Specialization | **Week:** W21-W22 | **Domain:** Advanced | **Flag:** —

## Brief

File 1 covered GKE vs. EKS as two single-cloud Kubernetes offerings. This file covers a third single-cloud offering (AKS) and then the actual multi-cluster-management layer this block is really about: **Anthos** and **Azure Arc** — the two production answers to "I have clusters on three different clouds/on-prem, how do I manage policy, GitOps, and observability across all of them from one place?" This is the part of the "avoid vendor lock-in" interview conversation that most candidates skip past with a vague "you'd use a service mesh" — Anthos and Arc are the concrete tooling.

## AKS — Azure's take on managed Kubernetes

- **Control plane is free** on the Free tier (no per-cluster hourly fee like EKS) — you only pay for nodes. A paid **Standard tier** exists for uptime SLA guarantees and higher API server scale limits, which most production clusters should use; Free tier has no SLA.
- **Node pools split into system and user pools.** The **system node pool** runs critical cluster add-ons (`coredns`, `metrics-server`, etc.) and should be tainted to keep user workloads off it; **user node pools** run your actual application workloads and can be added/scaled/deleted independently, including with different VM SKUs (e.g., a GPU pool alongside a general-purpose pool).
- **Virtual nodes (ACI-backed)** are AKS's Fargate-equivalent — pods scheduled onto Azure Container Instances instead of a VM node, giving fast burst-scale-out without pre-provisioning VM capacity (subject to ACI's own feature/network limitations, notably weaker for stateful/DaemonSet-style workloads).
- **Networking**: **kubenet** (legacy, pod IPs are NAT'd behind the node's IP — simpler, fewer VNet IPs consumed, but no direct pod routability and it doesn't support Windows nodes or some advanced features) vs. **Azure CNI** (pods get real VNet IPs, directly routable — the modern default, especially **Azure CNI Overlay**, which gives pods IPs from an overlay space instead of consuming VNet address space per pod, fixing Azure CNI's historical IP-exhaustion complaint while keeping direct routability for node-to-pod traffic).
- **Identity**: **Microsoft Entra Workload ID** (successor to the deprecated `aad-pod-identity` project) federates a Kubernetes ServiceAccount token with an Entra ID app registration via OIDC — the same shape as GKE Workload Identity and EKS IRSA.

```bash
# AKS cluster with Azure CNI Overlay + Workload Identity enabled
az aks create \
  --resource-group multicloud-demo-rg \
  --name multicloud-demo-aks \
  --node-count 2 \
  --network-plugin azure \
  --network-plugin-mode overlay \
  --enable-oidc-issuer \
  --enable-workload-identity \
  --generate-ssh-keys

az aks get-credentials --resource-group multicloud-demo-rg --name multicloud-demo-aks

# Add a dedicated user node pool
az aks nodepool add \
  --resource-group multicloud-demo-rg \
  --cluster-name multicloud-demo-aks \
  --name userpool \
  --node-count 2 \
  --mode User
```

## Anthos — Google's fleet-management answer

Anthos is Google's answer to "manage Kubernetes clusters everywhere, not just on GKE" — on-prem (Anthos on VMware, Anthos on Bare Metal), on other clouds (via **Attached Clusters**, the modern lightweight successor to the older Anthos on AWS/Azure offering), and on GKE itself. The core organizing concept is the **fleet**: a logical grouping of clusters registered under one GCP project, regardless of where they physically run.

- **Registering a cluster into a fleet** is the first step for anything Anthos does with it — an EKS or AKS cluster becomes "Anthos-attached" via `gcloud container fleet memberships register`, which installs a lightweight **Connect Agent** that maintains an outbound-only tunnel back to Google Cloud (no inbound firewall holes needed on the external cluster — the same "agent dials home" pattern Azure Arc uses).
- **Config Sync** (part of Anthos Config Management) is a GitOps operator: point it at a Git repo of Kubernetes manifests/Kustomize/Helm, and it continuously reconciles every registered cluster in the fleet to match — the multi-cluster equivalent of Argo CD/Flux, but fleet-aware out of the box (one repo, many clusters, optional per-cluster overlays via `ClusterSelector`).
- **Policy Controller** is Anthos's packaging of **OPA Gatekeeper** with a curated constraint template library (CIS benchmarks, Pod Security Standards) — applied fleet-wide, so a new cluster registering into the fleet inherits the same guardrails automatically.
- **Anthos Service Mesh (ASM)** is Google's managed Istio distribution — the control plane lifecycle (upgrades, cert rotation) is Google-managed even when the data plane spans clusters you operate yourself, which meaningfully lowers the operational burden of running multi-cluster Istio by hand (see file 3).

```bash
# Register an external (e.g., EKS) cluster into an Anthos fleet
gcloud container fleet memberships register eks-demo-membership \
  --context=eks-context \
  --kubeconfig=~/.kube/eks-config \
  --enable-workload-identity
```

## Azure Arc — Microsoft's take on "any cluster, any cloud"

Azure Arc-enabled Kubernetes solves a similar problem from the opposite architectural direction: instead of a Google-opinionated stack (fleet + ASM + Config Sync) layered onto external clusters, Arc's core move is projecting **any CNCF-conformant cluster** as an **Azure Resource Manager (ARM) resource** — meaning it shows up in the Azure Portal, gets an ARM resource ID, and becomes a target for Azure's existing governance tooling (Azure Policy, Azure Monitor, RBAC) as if it were a native Azure resource, even if it's an on-prem k3s cluster or a GKE cluster.

- **Connection mechanism**: `az connectedk8s connect` installs an agent that establishes an **outbound-only** connection to Azure (via Azure Relay) — same "no inbound holes" pattern as Anthos's Connect Agent.
- **Extensions** are how capability gets added post-connection: the **Flux (GitOps) extension** (`Microsoft.KubernetesConfiguration`) for GitOps reconciliation, **Azure Policy for Kubernetes** (Gatekeeper-based, same OPA foundation as Anthos Policy Controller) for admission-time guardrails, and **Azure Monitor Container Insights** for centralized logs/metrics from non-Azure clusters flowing into the same Log Analytics workspace as native AKS clusters.
- **Custom Locations** extends this further — Arc-enabled clusters can host Azure PaaS-like services (Arc-enabled App Service, Arc-enabled data services) on infrastructure you control, which is Azure's specific "run our managed-service experience anywhere" play.

```bash
az connectedk8s connect \
  --name gke-demo-arc \
  --resource-group multicloud-demo-rg \
  --kube-config ~/.kube/gke-config \
  --kube-context gke-context

# Attach the Flux GitOps extension to the newly connected cluster
az k8s-configuration flux create \
  --name gke-demo-gitops \
  --cluster-name gke-demo-arc \
  --resource-group multicloud-demo-rg \
  --cluster-type connectedClusters \
  --url https://github.com/your-org/fleet-config \
  --branch main \
  --kustomization name=infra path=./clusters/gke-demo prune=true
```

## Anthos vs. Azure Arc — the actual architectural difference

Both register external clusters via an outbound-only agent and both layer GitOps + policy + monitoring on top — the pattern is converging industry-wide. The difference worth stating precisely in an interview:

- **Anthos** brings a specific, opinionated, Google-built stack (Config Sync, Policy Controller, ASM) to every cluster it touches — you're adopting Google's toolchain for GitOps and mesh, regardless of where the cluster physically runs.
- **Azure Arc** brings the **ARM resource model and governance plane** to every cluster — Azure Policy, Azure RBAC, Azure Monitor, Cost Management — and treats GitOps (Flux) as one pluggable extension among several, not the mandatory backbone. Arc's pitch is closer to "make every cluster a first-class Azure-governed resource" than "make every cluster look like GKE."

Both exist for the same underlying business reason: **the multi-cluster/multi-cloud management-plane problem is real enough that every major cloud built a product for it**, and picking one over the other is often more about which cloud's IAM/policy/observability ecosystem you're already standardized on than a pure technical delta.

## Points to Remember

- AKS's control plane is free (Free tier) with no HA SLA; the Standard tier is the production-appropriate paid option — this is a real cost/reliability tradeoff to name explicitly, not just "AKS is cheaper."
- Azure CNI Overlay is the modern default that fixes classic Azure CNI's VNet-IP-exhaustion problem while keeping pods directly routable — know this exists before defaulting to kubenet out of habit.
- The fleet (Anthos) / Arc-enabled resource (Azure) concept is the actual unit of multi-cluster management — not the individual cluster. Both use an outbound-only "agent dials home" connection model so you never open inbound firewall rules to an external cluster.
- Anthos Policy Controller and Azure Policy for Kubernetes are both Gatekeeper/OPA under the hood — the differentiation is in the curated constraint libraries and how policies are authored/distributed, not the underlying enforcement engine.
- Anthos = Google's opinionated toolchain projected everywhere; Azure Arc = Azure's governance/ARM model projected everywhere. Neither is "the" multi-cluster answer — the right choice tracks whichever cloud's control/policy plane you're already standardized on.

## Common Mistakes

- Defaulting to AKS's kubenet network plugin "because it's simpler" without knowing Azure CNI Overlay now solves the IP-exhaustion problem that made kubenet attractive in the first place — kubenet also blocks some newer AKS features and Windows node pools.
- Assuming Anthos only works on GKE — Attached Clusters lets you register EKS, AKS, and on-prem clusters into a fleet; the fleet/Config Sync/Policy Controller story is explicitly meant to span clouds.
- Confusing Azure Arc with AKS — Arc-enabled Kubernetes is a governance/connectivity layer over *any* cluster (including non-Azure ones); it is not a Kubernetes distribution or a hosting service itself.
- Treating Policy Controller/Azure Policy for Kubernetes as a replacement for network-level security controls — admission-time policy (what gets deployed) and network policy/segmentation (how deployed workloads can talk to each other) are complementary layers, not substitutes.
- Registering a cluster into a fleet or connecting it via Arc and stopping there — the value only shows up once GitOps, policy, and monitoring are actually layered on top; a "registered but bare" cluster is just a database entry.
