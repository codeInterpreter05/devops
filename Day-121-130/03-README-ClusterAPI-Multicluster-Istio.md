# Day 121-130 — Multi-Cloud & Edge III: Cluster API and Multi-Cluster Istio

**Phase:** 4 – Advanced/Specialization | **Week:** W21-W22 | **Domain:** Advanced | **Flag:** —

## Brief

Files 1 and 2 covered *provider-native* ways to run and register clusters (gcloud/az/eksctl, fleets, Arc). This file covers the two pieces of tooling that let you treat cluster lifecycle and cross-cluster networking as **declarative, provider-agnostic Kubernetes APIs** rather than a pile of cloud-specific CLI invocations: **Cluster API** (create/scale/upgrade/delete clusters across any supported infrastructure provider using the same YAML shape) and **multi-cluster Istio** (route traffic and enforce mTLS across cluster boundaries as if they were one mesh). This is the "Best Resource" file for the block — Cluster API's own documentation is the assigned primary resource (see RESOURCES.md).

## Cluster API — clusters as Kubernetes objects, not CLI incantations

**Cluster API (CAPI)** is a Kubernetes sub-project that lets you manage the *lifecycle* of Kubernetes clusters — create, scale, upgrade, delete — using Kubernetes-native declarative APIs (`kubectl apply`, controllers, reconciliation loops) instead of `gcloud container clusters create` / `eksctl create cluster` / `az aks create` as three unrelated imperative tools. The mental model: you already trust Kubernetes controllers to reconcile a `Deployment`'s desired replica count — CAPI extends that same trust to "how many worker nodes does this workload cluster have, and on what cloud."

### The core objects

- **Management cluster** — a (usually small, sometimes throwaway) Kubernetes cluster that runs the CAPI controllers. It doesn't run your workloads; its job is to *create and manage other clusters*.
- **Workload clusters** — the clusters CAPI actually provisions for real work, defined declaratively and reconciled by the management cluster's controllers.
- **`Cluster`** — the top-level resource tying together a control-plane reference and an infrastructure reference for one workload cluster.
- **`InfrastructureCluster`** (provider-specific: `AWSCluster`, `AzureCluster`, `GCPCluster`, `VSphereCluster`, ...) — the cloud-specific networking/VPC scaffolding for that cluster, owned by an **infrastructure provider**.
- **`MachineDeployment` / `MachineSet` / `Machine`** — the exact same Deployment/ReplicaSet/Pod pattern you already know, except a `Machine` represents a **VM** (a control-plane or worker node) instead of a container. Scaling a node pool is `kubectl scale machinedeployment` or editing `replicas`, not a cloud-specific "resize node group" call.
- **Bootstrap provider** (commonly `kubeadm`) — turns a freshly created VM into a functioning Kubernetes node by generating and injecting the right `kubeadm init`/`kubeadm join` configuration.
- **`MachineHealthCheck`** — a controller that watches `Machine`/`Node` health and triggers automatic replacement of unhealthy nodes, independent of the cloud's own auto-healing (or lack of it).

### Setting it up — a real, provider-agnostic lifecycle

```bash
# 1. Any Kubernetes cluster can be the bootstrap management cluster — kind is standard for this
kind create cluster --name capi-mgmt

# 2. Install clusterctl (the CAPI CLI), then initialize whichever infra providers you need
clusterctl init --infrastructure aws,gcp

# 3. Generate a workload-cluster manifest for AWS (clusterawsadm handles credential encoding first)
export AWS_B64ENCODED_CREDENTIALS=$(clusterawsadm bootstrap credentials encode-as-profile)
clusterctl generate cluster capi-aws-demo \
  --infrastructure aws \
  --kubernetes-version v1.29.0 \
  --control-plane-machine-count 1 \
  --worker-machine-count 2 \
  > capi-aws-demo.yaml

kubectl apply -f capi-aws-demo.yaml

# 4. Watch it come up, then pull its kubeconfig
clusterctl describe cluster capi-aws-demo
clusterctl get kubeconfig capi-aws-demo > capi-aws-demo.kubeconfig
kubectl --kubeconfig capi-aws-demo.kubeconfig get nodes

# 5. The exact same pattern, on a different cloud — this is the actual "federation" payoff
clusterctl generate cluster capi-gcp-demo \
  --infrastructure gcp \
  --kubernetes-version v1.29.0 \
  --control-plane-machine-count 1 \
  --worker-machine-count 2 \
  > capi-gcp-demo.yaml
kubectl apply -f capi-gcp-demo.yaml
```

Scaling either cluster is a single `kubectl` operation against the **management** cluster, regardless of which cloud the workload cluster lives on:

```bash
kubectl scale machinedeployment capi-aws-demo-md-0 --replicas=4
```

### Why this is "federation," concretely

The interview-relevant point: CAPI doesn't create a single mesh cluster spanning clouds — it creates **independent clusters on independent clouds using one consistent API and one consistent lifecycle model**. The "federation" is at the *management/tooling* layer (one control point, one YAML shape, one `clusterctl` CLI, portable across AWS/GCP/Azure/vSphere/bare metal), not at the runtime networking layer — that's what multi-cluster Istio (below) is for. Conflating the two is a common mistake: CAPI answers "how do I consistently create/scale/upgrade clusters everywhere," multi-cluster Istio answers "how do services on those different clusters actually talk to each other."

A **pivot** (`clusterctl move`) is the other CAPI-specific concept worth knowing: you can bootstrap with a throwaway local `kind` management cluster, then move the CAPI custom resources (and therefore management responsibility) onto one of the workload clusters itself — so the temporary bootstrap cluster can be torn down once a "real" self-hosting management cluster exists.

## Multi-cluster Istio — making cross-cluster calls look local

Once you have multiple independent clusters (via CAPI, via GKE+EKS directly, whatever), services on Cluster A calling services on Cluster B needs: service discovery that spans clusters, mTLS that both clusters trust, and load-balancing behavior that's aware of *where* an endpoint physically lives (so you don't route latency-sensitive traffic across a continent when a local replica exists).

### The two topology models

- **Multi-Primary**: every cluster runs its **own** `istiod` control plane, and each control plane is given visibility into the other clusters' API servers (via a **remote secret**) so it can discover endpoints across all of them. No single cluster is a dependency for another's control plane to function — more resilient, more operationally symmetric.
- **Primary-Remote**: only some clusters run `istiod`; **remote** clusters have no local control plane and are configured (via `istioctl install` with an `externalIstiod` reference) to be managed by a primary cluster's `istiod` over the network. Simpler control-plane footprint, but remote clusters now depend on the primary's control plane being reachable.

Both models additionally split on **network topology**:
- **Single network** (flat, routable pod-to-pod IPs across clusters — rare across real clouds without VPN/peering) needs no gateway; pods can reach each other's IPs directly.
- **Multi-network** (the realistic cross-cloud case — GKE and EKS pods are never on the same flat routable network) requires an **east-west gateway** on each cluster: an Istio `Gateway` dedicated to routing *inter-cluster* mesh traffic (mutually authenticated via mTLS), so cross-cluster calls go pod → east-west gateway → east-west gateway → pod instead of needing direct pod routability.

### Setting up multi-primary, multi-network (the realistic GKE+EKS case)

```bash
# On cluster 1 (GKE) — install with a shared meshID, distinct network
istioctl install -f - <<EOF
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: demo
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: gke-cluster
      network: network1
EOF

# Deploy the east-west gateway on cluster 1
samples/multicluster/gen-eastwest-gateway.sh --network network1 | \
  istioctl install --context=gke-context -y -f -

# On cluster 2 (EKS) — same meshID, DIFFERENT network
istioctl install --context=eks-context -f - <<EOF
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: demo
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: eks-cluster
      network: network2
EOF
samples/multicluster/gen-eastwest-gateway.sh --network network2 | \
  istioctl install --context=eks-context -y -f -

# Exchange remote secrets so each control plane can discover the other's endpoints
istioctl x create-remote-secret --context=gke-context --name=gke-cluster | \
  kubectl apply -f - --context=eks-context
istioctl x create-remote-secret --context=eks-context --name=eks-cluster | \
  kubectl apply -f - --context=gke-context
```

### Shared trust — the part that actually makes cross-cluster mTLS work

None of the above produces working cross-cluster mTLS unless **every cluster's `istiod` signs workload certificates from the same root CA** — this is the **shared trust domain** requirement. In a lab you'd generate one root cert and intermediate certs per cluster from it before installing Istio on any cluster (Istio's `samples/certs` walk-through does exactly this); in production, teams commonly delegate this to **cert-manager + `istio-csr`** so certificate issuance is centrally managed and auditable rather than a one-time manual cert-generation step nobody remembers how to redo. Skipping this and installing Istio on each cluster with its own self-generated root CA is the single most common reason "multi-cluster Istio" demos fail — each cluster mTLS-authenticates fine *internally* and then silently rejects the other cluster's workload certs at the east-west gateway.

### Locality-aware routing — keeping traffic local when a local replica exists

Istio derives a workload's **locality** (region/zone/sub-zone) from standard Kubernetes node topology labels (`topology.kubernetes.io/region`, `topology.kubernetes.io/zone`) — this is what lets a `DestinationRule` prefer same-region/same-zone endpoints and only fail over to another cluster/region when local capacity is actually unhealthy:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: hello-multicluster
spec:
  host: hello.default.svc.cluster.local
  trafficPolicy:
    outlierDetection:
      consecutive5xxErrors: 3
      interval: 10s
      baseEjectionTime: 30s
    loadBalancer:
      localityLbSetting:
        enabled: true
        failover:
          - from: us-central1
            to: us-east-1
```

`outlierDetection` is what actually triggers failover — without it configured, `localityLbSetting` has nothing to react to and traffic just stays pinned locally even during a real local outage. This pairing (locality preference + outlier-detection-driven failover) is the concrete mechanism behind "active-active multi-region/multi-cloud with automatic failover," a phrase that shows up in a lot of architecture-diagram slides with no explanation of how it's actually implemented.

## Points to Remember

- Cluster API standardizes *cluster lifecycle* (create/scale/upgrade/delete) as Kubernetes-native declarative objects across infrastructure providers — it does not, by itself, connect the resulting clusters' networks or service meshes together.
- The management cluster runs CAPI's controllers and is not where your workloads run; `clusterctl move` (pivot) lets you migrate management responsibility off a throwaway bootstrap cluster onto a real one.
- `Machine`/`MachineSet`/`MachineDeployment` intentionally mirror `Pod`/`ReplicaSet`/`Deployment` — scaling a node pool across any CAPI-supported cloud is the same `kubectl scale` operation.
- Multi-cluster Istio needs three things to actually work across clouds: a shared meshID, a per-cluster network identifier plus east-west gateways (for the realistic multi-network case), and — most commonly missed — a **shared root CA** across every cluster's `istiod` for cross-cluster mTLS to succeed.
- Locality-aware routing needs both a locality-preference config (`localityLbSetting`) *and* `outlierDetection` — the first prefers local traffic, the second is what actually detects failure and triggers cross-cluster failover.

## Common Mistakes

- Assuming CAPI creates one unified cluster spanning clouds — it creates and manages independent clusters through one consistent API; cross-cluster runtime networking is a separate problem (Istio, Submariner, cloud VPC peering, etc.), not something CAPI solves.
- Installing Istio independently on each cluster with each cluster's own self-signed root CA, then being confused when cross-cluster calls fail mTLS verification at the east-west gateway — the root CA must be shared/chained across every cluster in the mesh.
- Configuring `localityLbSetting.failover` without `outlierDetection` and expecting automatic failover to "just work" — without outlier detection there's no failure signal driving the failover decision.
- Treating Primary-Remote as strictly "simpler/better" — it introduces a real dependency (remote clusters can't function without the primary's control plane reachable), which is a legitimate reason many production multi-cloud setups choose Multi-Primary despite its larger control-plane footprint.
- Forgetting that a CAPI management cluster is itself a piece of infrastructure that needs backup/DR planning — if it's lost without a recent etcd backup or a completed pivot, you lose your only record of what workload clusters exist and how they were configured (though the workload clusters themselves keep running).
