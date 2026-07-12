# Day 53 — Lab: Phase 1 Project — Full Stack IaC

**Goal:** Build, deploy, and document a complete end-to-end production-shaped EKS environment — the assigned hands-on activity — integrating Terraform, Helm, ArgoCD, External Secrets, and autoscaling into one working stack with a comprehensive README.

**Prerequisites:** Everything from Phase 1: an AWS account (sandbox), Terraform, `kubectl`, `helm`, `argocd` CLI, an existing GitHub repo to push to. Budget real AWS cost for this lab (EKS control plane + RDS + ElastiCache + NAT Gateways are not free-tier) — plan to tear it all down promptly after.

---

### Lab 1 — Provision the infrastructure layer

1. Write (or adapt from Day 53's file 1) a Terraform root module composing: VPC module, EKS module, RDS instance, ElastiCache replication group, and the IAM/IRSA roles needed for the Load Balancer Controller, External Secrets, and Karpenter.
2. Plan and apply in stages, verifying each layer before moving to the next:
   ```bash
   terraform init
   terraform plan -target=module.vpc
   terraform apply -target=module.vpc
   terraform plan -target=module.eks
   terraform apply -target=module.eks
   terraform apply   # remaining resources (RDS, ElastiCache, IAM)
   ```
3. Confirm cluster access:
   ```bash
   aws eks update-kubeconfig --name prod-eks-cluster --region us-east-1
   kubectl get nodes
   ```

**Success criteria:** `kubectl get nodes` shows healthy nodes, and `terraform state list` shows every layer (VPC, EKS, RDS, ElastiCache, IAM roles) provisioned successfully.

---

### Lab 2 — Bootstrap ArgoCD and deploy via GitOps

1. Install ArgoCD into the cluster:
   ```bash
   kubectl create namespace argocd
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   kubectl port-forward svc/argocd-server -n argocd 8080:443 &
   ```
2. Push a Helm chart for a simple app (or reuse any app from earlier phases) to a Git repo, then create an ArgoCD `Application` pointing at it (as in file 2's example), with `automated: { prune: true, selfHeal: true }`.
3. Apply the Application and confirm ArgoCD syncs it:
   ```bash
   kubectl apply -f argocd-app.yaml
   argocd app get my-app
   argocd app sync my-app
   ```
4. Prove `selfHeal` works: manually scale the Deployment via `kubectl scale`, then watch ArgoCD revert it back to the replica count declared in Git within moments.

**Success criteria:** The app is running, fully declared by Git, and a manual `kubectl` change to a synced resource gets automatically reverted by ArgoCD.

---

### Lab 3 — Wire up External Secrets Operator

1. Store the RDS connection details as a secret in AWS Secrets Manager (or confirm the RDS-managed one already exists via `manage_master_user_password`).
2. Install External Secrets Operator via Helm, install/configure a `SecretStore` bound to an IRSA service account, and deploy an `ExternalSecret` referencing the RDS credential.
3. Confirm the Kubernetes Secret was created and mount it into the app Deployment as an environment variable.

**Success criteria:** The app can connect to RDS using credentials that never appear anywhere in your Git repo, Helm values, or Terraform state as plaintext.

---

### Lab 4 — Wire up KEDA and Karpenter

1. Install KEDA and Karpenter via Helm (both are typically bootstrapped alongside ArgoCD apps in a real setup, but a direct `helm install` is fine for this lab).
2. Deploy a `ScaledObject` targeting the SQS queue from Day 52's lab (or a simple CPU-based scaler if you skipped that lab), with `minReplicaCount: 0`.
3. Deploy a Karpenter `NodePool` + `EC2NodeClass`.
4. Trigger load (send a batch of SQS messages, or run a CPU-stress job) and watch pods scale up, then watch `kubectl get nodes` as Karpenter provisions new capacity to fit them.

**Success criteria:** You observe, via `kubectl get pods -w` and `kubectl get nodes -w` side by side, the full chain: event arrives → KEDA scales pods → pods go Pending → Karpenter provisions a node → pods schedule and run → load clears → pods scale to zero → node is consolidated away.

---

### Lab 5 — Write the comprehensive README (core hands-on deliverable)

Write a README for this project repo covering: architecture diagram (ASCII is fine), the reasoning behind each layer's design decisions (subnetting, IRSA scoping, GitOps vs. push deploys, secrets flow, autoscaling chain), how to reproduce it from scratch, and known limitations/what you'd add for a real production rollout (multi-region, WAF, GuardDuty/Security Hub from Day 49, CI pipeline feeding the Git repo ArgoCD watches).

**Success criteria:** A colleague unfamiliar with the project could read the README and understand the full architecture and its rationale without needing to ask you anything.

---

### Cleanup

```bash
argocd app delete my-app
kubectl delete namespace argocd
terraform destroy   # tear down everything — this is real, billed AWS infrastructure
```

### Stretch challenge

Push this entire project to a public (or private, your choice) GitHub repository with the README from Lab 5, and add a GitHub Actions workflow that runs `terraform plan` (not apply) on every pull request touching the `infra/` directory, posting the plan output as a PR comment — closing the loop between the IaC layer and the GitOps layer with a review gate.
