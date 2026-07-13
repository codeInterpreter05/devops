# Day 121-130 — Cheatsheet: Multi-Cloud & Edge

## GKE (gcloud) — cluster lifecycle

```bash
# Standard, regional, VPC-native cluster (production-shaped default)
gcloud container clusters create <name> \
  --region=<region> --release-channel=regular \
  --num-nodes=1 --machine-type=e2-medium \
  --enable-ip-alias \
  --workload-pool=<project>.svc.id.goog

# Autopilot (per-pod billing, Google manages nodes + policy)
gcloud container clusters create-auto <name> --region=<region>

gcloud container clusters get-credentials <name> --region=<region>
gcloud container clusters list
gcloud container clusters resize <name> --node-pool=<pool> --num-nodes=<n> --region=<region>
gcloud container clusters delete <name> --region=<region>

# Node pools (Standard mode only)
gcloud container node-pools create <pool-name> --cluster=<name> --region=<region> \
  --machine-type=e2-standard-4 --num-nodes=2
gcloud container node-pools list --cluster=<name> --region=<region>

# Workload Identity binding
gcloud iam service-accounts create <gsa-name>
gcloud iam service-accounts add-iam-policy-binding <gsa-name>@<project>.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:<project>.svc.id.goog[<namespace>/<ksa-name>]"
kubectl annotate serviceaccount <ksa-name> \
  iam.gke.io/gcp-service-account=<gsa-name>@<project>.iam.gserviceaccount.com
```

## EKS (eksctl / AWS CLI) — cluster lifecycle

```bash
# Managed node group cluster
eksctl create cluster --name <name> --region <region> --version 1.29 \
  --nodegroup-name <ng-name> --node-type t3.medium --nodes 2 --managed

aws eks update-kubeconfig --name <name> --region <region>
eksctl get cluster --region <region>
eksctl scale nodegroup --cluster <name> --name <ng-name> --nodes <n> --region <region>
eksctl delete cluster --name <name> --region <region>

# Fargate profile (EKS's closest analog to Autopilot's "no nodes" model, scoped per namespace)
eksctl create fargateprofile --cluster <name> --name <profile-name> --namespace <ns>

# IRSA (IAM Roles for Service Accounts)
eksctl utils associate-iam-oidc-provider --cluster <name> --approve
eksctl create iamserviceaccount \
  --cluster <name> --namespace <ns> --name <sa-name> \
  --attach-policy-arn <policy-arn> --approve

# Cluster / node group upgrades (two separate steps, unlike GKE's auto-upgrade)
eksctl upgrade cluster --name <name> --approve
eksctl upgrade nodegroup --cluster <name> --name <ng-name> --kubernetes-version <ver>
```

## AKS (az CLI) — cluster lifecycle

```bash
# Standard tier cluster, Azure CNI Overlay, Workload Identity enabled
az aks create \
  --resource-group <rg> --name <name> --node-count 2 \
  --network-plugin azure --network-plugin-mode overlay \
  --enable-oidc-issuer --enable-workload-identity \
  --tier standard \
  --generate-ssh-keys

az aks get-credentials --resource-group <rg> --name <name>
az aks list -o table
az aks scale --resource-group <rg> --name <name> --node-count <n>
az aks delete --resource-group <rg> --name <name>

# Node pools (system vs. user)
az aks nodepool add --resource-group <rg> --cluster-name <name> --name <pool> \
  --node-count 2 --mode User
az aks nodepool list --resource-group <rg> --cluster-name <name>

# Virtual nodes (ACI-backed, Fargate-equivalent)
az aks nodepool add --resource-group <rg> --cluster-name <name> --name virtualnp \
  --node-count 1 --node-vm-size Standard_D2_v2 --enable-virtual-node --os-type Linux
```

## Anthos / Azure Arc — fleet registration

```bash
# Anthos: register an external cluster into a fleet
gcloud container fleet memberships register <membership-name> \
  --context=<kube-context> --kubeconfig=<path> --enable-workload-identity
gcloud container fleet memberships list

# Azure Arc: connect any CNCF-conformant cluster
az connectedk8s connect --name <arc-name> --resource-group <rg> \
  --kube-config <path> --kube-context <context>

# Arc: attach the Flux GitOps extension
az k8s-configuration flux create \
  --name <config-name> --cluster-name <arc-name> --resource-group <rg> \
  --cluster-type connectedClusters \
  --url <git-repo-url> --branch main \
  --kustomization name=infra path=./clusters/<cluster> prune=true
```

## Cluster API (clusterctl)

```bash
clusterctl init --infrastructure aws,gcp,azure   # install providers into the mgmt cluster
clusterctl init list-providers

# AWS credential encoding (AWS-specific prerequisite)
export AWS_B64ENCODED_CREDENTIALS=$(clusterawsadm bootstrap credentials encode-as-profile)

clusterctl generate cluster <name> \
  --infrastructure <aws|gcp|azure> \
  --kubernetes-version v1.29.0 \
  --control-plane-machine-count 1 \
  --worker-machine-count 2 > <name>.yaml
kubectl apply -f <name>.yaml

clusterctl describe cluster <name>                 # watch provisioning status
clusterctl get kubeconfig <name> > <name>.kubeconfig

kubectl scale machinedeployment <name>-md-0 --replicas=<n>   # same op, any cloud

clusterctl move --to-kubeconfig <new-mgmt-kubeconfig>   # pivot management cluster
clusterctl delete cluster <name>
```

## Istio multi-cluster (istioctl)

```bash
# Per-cluster install with shared meshID, distinct network
istioctl install --context=<ctx> -y -f - <<EOF
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: demo
  values:
    global:
      meshID: mesh1
      multiCluster: { clusterName: <cluster-name> }
      network: <network-id>
EOF

# East-west gateway (multi-network case — the realistic cross-cloud scenario)
samples/multicluster/gen-eastwest-gateway.sh --network <network-id> | \
  istioctl install --context=<ctx> -y -f -

# Exchange remote secrets (run both directions)
istioctl x create-remote-secret --context=<ctx-a> --name=<cluster-a> | \
  kubectl apply -f - --context=<ctx-b>

# Verify mTLS and endpoint visibility
istioctl x describe pod <pod> --context=<ctx>
istioctl proxy-config endpoints <pod> --context=<ctx> | grep <remote-service>
```

## Locality-aware routing (DestinationRule snippet)

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: <service>
spec:
  host: <service>.<namespace>.svc.cluster.local
  trafficPolicy:
    outlierDetection:              # REQUIRED for failover to actually trigger
      consecutive5xxErrors: 3
      interval: 10s
      baseEjectionTime: 30s
    loadBalancer:
      localityLbSetting:
        enabled: true
        failover:
          - from: <region-a>
            to: <region-b>
```

## Cloudflare Workers (wrangler)

```bash
npm create cloudflare@latest <project>
wrangler dev              # local dev server
wrangler deploy            # ships globally, no region selection
wrangler tail               # live production log stream

wrangler kv:namespace create <name>
wrangler r2 bucket create <bucket>
wrangler r2 bucket create <bucket> --location <hint>
```

## Egress cost quick-reference (conceptual shape — rates change, know the pattern)

| Path | AWS | GCP | Azure | Cloudflare R2 |
|---|---|---|---|---|
| To internet (first tier) | ~$0.09/GB | ~$0.12/GB (varies by destination) | ~$0.087/GB | — |
| Inter-AZ / intra-region | ~$0.01/GB each way | Low/free intra-zone | Low/varies | — |
| Inter-region (same cloud) | ~$0.02/GB | ~$0.01–0.08/GB | Varies | — |
| Cross-cloud (AWS↔GCP↔Azure) | Billed as standard internet egress — **no inter-cloud discount tier** | Same | Same | N/A |
| Object storage egress | Standard tiered rate | Standard tiered rate | Standard tiered rate | **$0/GB** |
| Ingress (any direction) | Free | Free | Free | Free |
| NAT Gateway processing | ~$0.045/GB **+** hourly charge, **on top of** egress rate | Cloud NAT: per-GB processed, **on top of** egress rate | NAT Gateway: per-GB processed, **on top of** egress rate | N/A |

**Rule of thumb for interviews:** egress is metered and tiered on every cloud, ingress is always free, and moving data *between* clouds pays the full internet-egress rate on both sides with zero special discount — that asymmetry is the concrete mechanism behind "multi-cloud has hidden egress costs."
