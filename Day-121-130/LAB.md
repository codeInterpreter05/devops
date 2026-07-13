# Day 121-130 — Lab: Multi-Cloud & Edge Build-Out

**Goal:** Leave this 10-day block having actually deployed the same app to GKE and EKS, federated cluster lifecycle across clouds with Cluster API, wired a real multi-cluster Istio mesh between two clusters, and produced a filled-in networking/RBAC/cost comparison table backed by real numbers you measured yourself — not a guess.

**Prerequisites:** Active AWS, GCP, and (optional, for the AKS stretch) Azure accounts with billing alarms set (this lab spins up real, billed infrastructure). `gcloud`, `aws` CLI + `eksctl`, `az` (optional), `kubectl`, `helm`, `kind`, `clusterctl`, `istioctl`, and `wrangler` (for the optional Cloudflare stretch) installed locally. Budget roughly $15-40 in cloud spend for the full 10 days if you tear everything down promptly each night — see Cleanup.

This lab is a single 10-day arc across five parts. Do them in order — each part's comparison-table entries build on the previous one.

---

### Part 1 (Days 1-2) — Deploy to GKE

1. Set your project and enable the API:
   ```bash
   gcloud config set project <PROJECT_ID>
   gcloud services enable container.googleapis.com
   ```
2. Create a **regional, VPC-native Standard cluster**:
   ```bash
   gcloud container clusters create multicloud-demo \
     --region=us-central1 \
     --release-channel=regular \
     --num-nodes=1 \
     --machine-type=e2-medium \
     --enable-ip-alias \
     --workload-pool=$(gcloud config get-value project).svc.id.goog

   gcloud container clusters get-credentials multicloud-demo --region us-central1
   ```
3. Deploy the sample app and expose it:
   ```bash
   kubectl create deployment hello-gke --image=gcr.io/google-samples/hello-app:1.0
   kubectl expose deployment hello-gke --type=LoadBalancer --port=80 --target-port=8080
   kubectl get svc hello-gke -w   # wait for EXTERNAL-IP, then curl it
   ```
4. Inspect the networking model directly — don't just take file 1's word for it:
   ```bash
   kubectl get nodes -o wide                          # note node internal IPs (VPC subnet range)
   kubectl get pod -o wide -l app=hello-gke            # note pod IP (alias/secondary range, still VPC-routable)
   kubectl describe node <node-name> | grep -A3 "Allocatable"   # note max pods per node
   ```
5. Set up Workload Identity end to end for the RBAC comparison later:
   ```bash
   gcloud iam service-accounts create hello-gke-gsa
   kubectl create serviceaccount hello-gke-ksa
   gcloud iam service-accounts add-iam-policy-binding hello-gke-gsa@$(gcloud config get-value project).iam.gserviceaccount.com \
     --role roles/iam.workloadIdentityUser \
     --member "serviceAccount:$(gcloud config get-value project).svc.id.goog[default/hello-gke-ksa]"
   kubectl annotate serviceaccount hello-gke-ksa \
     iam.gke.io/gcp-service-account=hello-gke-gsa@$(gcloud config get-value project).iam.gserviceaccount.com
   ```
6. Try Autopilot side by side so you have a real comparison point for file 1's Autopilot-vs-Standard section:
   ```bash
   gcloud container clusters create-auto multicloud-demo-autopilot --region=us-central1
   ```
   Attempt to deploy a privileged pod on the Autopilot cluster and confirm it's rejected by policy — this is the concrete evidence behind "Autopilot enforces baseline security policy," not just a claim from the README.

**Success criteria:** `curl` against the GKE LoadBalancer's external IP returns the hello-app response; you can state the pod's actual IP range and confirm it's VPC-routable (not NAT'd/overlay); Workload Identity binding succeeds (`kubectl describe sa hello-gke-ksa` shows the annotation); a privileged pod is demonstrably rejected on the Autopilot cluster.

---

### Part 2 (Days 3-4) — Deploy to EKS and compare

1. Create a managed-node-group EKS cluster:
   ```bash
   eksctl create cluster \
     --name multicloud-demo \
     --region us-east-1 \
     --version 1.29 \
     --nodegroup-name standard-workers \
     --node-type t3.medium \
     --nodes 2 \
     --managed

   aws eks update-kubeconfig --name multicloud-demo --region us-east-1
   ```
2. Deploy the **same image** used on GKE, exposed the same way:
   ```bash
   kubectl create deployment hello-eks --image=gcr.io/google-samples/hello-app:1.0
   kubectl expose deployment hello-eks --type=LoadBalancer --port=80 --target-port=8080
   kubectl get svc hello-eks -w
   ```
3. Repeat the same networking inspection as Part 1, step 4, on this cluster — this is the actual comparison, not a rehearsal:
   ```bash
   kubectl get nodes -o wide
   kubectl get pod -o wide -l app=hello-eks
   kubectl describe node <node-name> | grep -A3 "Allocatable"
   ```
   Specifically check the max-pods number against GKE's — this is where EKS's ENI-based pod-IP ceiling on `t3.medium` becomes a concrete, measured number instead of an abstract claim.
4. Set up IRSA for the RBAC comparison:
   ```bash
   eksctl utils associate-iam-oidc-provider --cluster multicloud-demo --approve
   eksctl create iamserviceaccount \
     --cluster multicloud-demo --namespace default --name hello-eks-sa \
     --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
     --approve
   ```
5. **Fill in the first half of your comparison table** (template below) using what you just measured on both clusters.

**Success criteria:** `curl` against the EKS LoadBalancer's external IP (an AWS NLB/ELB hostname) returns the hello-app response; you have real, measured max-pods-per-node numbers for both clouds side by side; IRSA is confirmed working (`aws sts get-caller-identity` from inside a pod using `hello-eks-sa` returns the assumed role, not the node's role).

**Comparison table — fill in as you go (this is a deliverable, not decoration):**

| Dimension | GKE | EKS | AKS (if you do the stretch) |
|---|---|---|---|
| Control plane cost/mo | | | |
| Default CNI | | | |
| Pod IP source | | | |
| Measured max pods/node | | | |
| Identity mechanism | | | |
| LB type provisioned | | | |
| Time to cluster-ready (mm:ss) | | | |
| Egress path for private nodes | | | |

---

### Part 3 (Days 5-6) — Cluster API federation

1. Bootstrap a local management cluster:
   ```bash
   kind create cluster --name capi-mgmt
   clusterctl init --infrastructure aws,gcp
   ```
2. Generate and apply an AWS workload cluster via CAPI (note: this is a **different** AWS cluster from Part 2's — CAPI is provisioning it declaratively rather than via `eksctl`):
   ```bash
   export AWS_B64ENCODED_CREDENTIALS=$(clusterawsadm bootstrap credentials encode-as-profile)
   clusterctl generate cluster capi-aws-demo \
     --infrastructure aws \
     --kubernetes-version v1.29.0 \
     --control-plane-machine-count 1 \
     --worker-machine-count 2 \
     > capi-aws-demo.yaml
   kubectl apply -f capi-aws-demo.yaml
   clusterctl describe cluster capi-aws-demo   # watch until Ready
   ```
3. Do the same for GCP:
   ```bash
   clusterctl generate cluster capi-gcp-demo \
     --infrastructure gcp \
     --kubernetes-version v1.29.0 \
     --control-plane-machine-count 1 \
     --worker-machine-count 2 \
     > capi-gcp-demo.yaml
   kubectl apply -f capi-gcp-demo.yaml
   clusterctl describe cluster capi-gcp-demo
   ```
4. Pull both workload clusters' kubeconfigs from the **single** management cluster and confirm both are independently reachable:
   ```bash
   clusterctl get kubeconfig capi-aws-demo > capi-aws-demo.kubeconfig
   clusterctl get kubeconfig capi-gcp-demo > capi-gcp-demo.kubeconfig
   kubectl --kubeconfig capi-aws-demo.kubeconfig get nodes
   kubectl --kubeconfig capi-gcp-demo.kubeconfig get nodes
   ```
5. Prove the "one API, two clouds" point by scaling both the same way:
   ```bash
   kubectl scale machinedeployment capi-aws-demo-md-0 --replicas=3
   kubectl scale machinedeployment capi-gcp-demo-md-0 --replicas=3
   ```

**Success criteria:** Two independent workload clusters, on two different clouds, both created and scaled from the same management cluster using the identical `kubectl scale machinedeployment` command — you can point at the command and say "this is the same operation regardless of which cloud it's targeting," which is the actual point of Cluster API.

---

### Part 4 (Days 7-8) — Multi-cluster Istio

Use the two clusters from Part 1 (GKE) and Part 2 (EKS) — real clusters on different clouds, the realistic multi-network case.

1. Generate a shared root CA and per-cluster intermediate certs (do this **before** installing Istio anywhere — see file 3's "shared trust" section for why):
   ```bash
   # Following Istio's samples/certs walkthrough — generates root-cert.pem plus
   # a distinct ca-cert.pem/ca-key.pem intermediate per cluster, signed by the same root
   ```
2. Install Istio on GKE with a distinct network identifier:
   ```bash
   istioctl install --context=gke-context -y -f - <<EOF
   apiVersion: install.istio.io/v1alpha1
   kind: IstioOperator
   spec:
     profile: demo
     values:
       global:
         meshID: mesh1
         multiCluster: { clusterName: gke-cluster }
         network: network1
   EOF
   samples/multicluster/gen-eastwest-gateway.sh --network network1 | \
     istioctl install --context=gke-context -y -f -
   ```
3. Install Istio on EKS the same way, with `network2` and `eks-cluster`, plus its own east-west gateway.
4. Exchange remote secrets both directions (see file 3 for the exact commands) and confirm each cluster can see the other's endpoints:
   ```bash
   istioctl x create-remote-secret --context=gke-context --name=gke-cluster | \
     kubectl apply -f - --context=eks-context
   istioctl x create-remote-secret --context=eks-context --name=eks-cluster | \
     kubectl apply -f - --context=gke-context
   ```
5. Deploy the standard Istio multicluster `helloworld`/`sleep` samples split across both clusters (v1 on GKE, v2 on EKS) and confirm cross-cluster load balancing:
   ```bash
   kubectl --context=gke-context exec deploy/sleep -- curl -s helloworld.sample:5000/hello
   # run it several times — responses should alternate between "version: v1" (local) and
   # "version: v2" (across the east-west gateway to EKS)
   ```
6. Apply the locality-aware `DestinationRule` from file 3 and simulate a local failure (scale the local `helloworld` deployment to 0) to confirm traffic fails over to the other cluster instead of just erroring.

**Success criteria:** A `curl` from a pod on GKE reaches a service instance running on EKS (and vice versa) through the east-west gateways with mTLS intact (`istioctl x describe pod` confirms mTLS STRICT); scaling the local deployment to 0 causes traffic to visibly fail over to the remote cluster rather than failing outright.

---

### Part 5 (Days 9-10) — Cost analysis, and the Cloudflare/egress stretch

1. Pull real billing data for what you've actually spent so far this block:
   ```bash
   # GCP
   gcloud billing accounts list
   gcloud alpha billing accounts get-spend-information --billing-account=<ACCOUNT_ID>   # or use the Cost Table export

   # AWS
   aws ce get-cost-and-usage \
     --time-period Start=$(date -v-10d +%Y-%m-%d),End=$(date +%Y-%m-%d) \
     --granularity DAILY \
     --metrics "UnblendedCost" \
     --group-by Type=DIMENSION,Key=SERVICE
   ```
2. Specifically isolate **data transfer / egress** line items from the rest — this is the number the block is really testing whether you can produce:
   ```bash
   aws ce get-cost-and-usage \
     --time-period Start=$(date -v-10d +%Y-%m-%d),End=$(date +%Y-%m-%d) \
     --granularity DAILY \
     --metrics "UnblendedCost" \
     --filter '{"Dimensions":{"Key":"USAGE_TYPE_GROUP","Values":["EC2: Data Transfer"]}}'
   ```
3. Complete the comparison table from Part 2 with real cost figures now that you have several days of billing data for both control planes, node pools, and load balancers.
4. **Cloudflare Workers + R2 stretch** (optional but recommended — this is the concrete evidence behind file 4's "R2 has zero egress" claim):
   ```bash
   npm create cloudflare@latest egress-demo
   cd egress-demo
   wrangler r2 bucket create egress-demo-bucket
   # upload a multi-hundred-MB test object, then download it repeatedly from a
   # different network/region and confirm your Cloudflare bill shows $0 egress
   # for those downloads, unlike the S3/GCS equivalent would
   wrangler deploy
   ```
5. Write up a short, real cost narrative (a few sentences, not just the table) answering: which cloud's control plane was cheapest for this workload, which had the cheapest egress path for the Part 4 cross-cluster Istio traffic, and what the NAT Gateway/LB processing fees actually came to versus what you'd have guessed before doing the measurement.

**Success criteria:** A comparison table with every cell filled from **measured** data, not estimates; a short written cost narrative naming the actual cheapest/most-expensive dimension for this specific workload; (if you did the stretch) a Cloudflare billing screenshot or CLI output showing $0 egress for the R2 downloads.

---

## Cleanup

Do this at the end of **every session**, not just at the end of the block — clusters, load balancers, and NAT gateways all bill by the hour whether you're using them or not.

```bash
# GKE
gcloud container clusters delete multicloud-demo --region us-central1 --quiet
gcloud container clusters delete multicloud-demo-autopilot --region us-central1 --quiet

# EKS (also tears down the managed node group and associated ELB/NLB)
eksctl delete cluster --name multicloud-demo --region us-east-1

# Cluster API workload clusters, then the management cluster
kubectl delete cluster capi-aws-demo
kubectl delete cluster capi-gcp-demo
kind delete cluster --name capi-mgmt

# Istio (on each cluster before deleting the cluster itself, if not already torn down above)
istioctl uninstall --purge -y --context=gke-context
istioctl uninstall --purge -y --context=eks-context

# Cloudflare stretch
wrangler r2 bucket delete egress-demo-bucket
```

Before tearing anything down, explicitly check for orphaned billable resources both clouds are known to leave behind:

```bash
gcloud compute forwarding-rules list      # orphaned GCP load balancers
gcloud compute addresses list             # orphaned reserved IPs
aws elbv2 describe-load-balancers         # orphaned AWS load balancers
aws ec2 describe-nat-gateways             # orphaned NAT gateways — these bill hourly even idle
```

## What You Learned — final checklist

- [ ] I deployed the same container image to both GKE and EKS and can state, from measured data (not memory), the difference in max-pods-per-node caused by their CNI models.
- [ ] I set up Workload Identity (GKE) and IRSA (EKS) end to end and can explain the trust-federation mechanism behind each, not just that "both avoid static keys."
- [ ] I provisioned clusters on two different clouds from a single Cluster API management cluster and scaled both with the identical `kubectl scale machinedeployment` command.
- [ ] I have a working multi-cluster Istio mesh between GKE and EKS with mTLS verified, and I watched locality-aware failover actually happen when I killed the local endpoint.
- [ ] I have a comparison table filled with real measured numbers (not guesses) for networking, RBAC/identity, and cost across at least GKE and EKS.
- [ ] I can produce a concrete dollar estimate for cross-cloud egress on a stated workload size, and name at least two "hidden" cost line items (NAT processing, LB data processing) beyond the headline per-GB egress rate.
- [ ] Every cluster, load balancer, NAT gateway, and reserved IP created during this block has been confirmed deleted — I checked, not assumed.

If more than one or two boxes are unchecked, that's the specific gap to close before treating this block as done — this is a hands-on-proof block, and "I read about it" doesn't satisfy any of these criteria.
