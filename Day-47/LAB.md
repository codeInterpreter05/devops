# Day 47 — Lab: Terraform on AWS — Full Project

**Goal:** Provision a full EKS cluster with Terraform using the community module ecosystem, then deploy a sample app via Helm through CI — the complete pipeline this day's notes describe end to end.

**Prerequisites:** AWS CLI configured with sufficient permissions (VPC, EKS, IAM, S3, DynamoDB), Terraform >= 1.6 installed, `kubectl` and `helm` installed, a GitHub repository you control (for Lab 4). Expect real AWS charges (EKS control plane is ~$0.10/hr plus node costs) — use a sandbox account and tear down promptly.

---

### Lab 1 — Remote state backend, provisioned first

1. Bootstrap the state backend itself (this genuinely has to be created outside the main config, or via a separate tiny Terraform config applied once with local state):
   ```bash
   aws s3api create-bucket --bucket my-tf-state-$(whoami) --region us-east-1
   aws s3api put-bucket-versioning --bucket my-tf-state-$(whoami) --versioning-configuration Status=Enabled
   aws s3api put-bucket-encryption --bucket my-tf-state-$(whoami) \
     --server-side-encryption-configuration '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'
   aws dynamodb create-table --table-name terraform-locks \
     --attribute-definitions AttributeName=LockID,AttributeType=S \
     --key-schema AttributeName=LockID,KeyType=HASH \
     --billing-mode PAY_PER_REQUEST
   ```
2. Configure your root module's backend to point at this bucket/table (see Cheatsheet for the block).

**Success criteria:** `terraform init` succeeds against the S3 backend with no errors, and `aws dynamodb scan --table-name terraform-locks` shows an item appear during a `plan`/`apply` and disappear after it completes.

---

### Lab 2 — Full EKS cluster in Terraform (the assigned hands-on activity, part 1)

1. Create a project structure:
   ```
   environments/dev/main.tf
   environments/dev/variables.tf
   environments/dev/outputs.tf
   environments/dev/backend.tf
   ```
2. In `main.tf`, compose the VPC and EKS modules (see 01-README for the exact module blocks) — pin both to specific versions.
3. `terraform init && terraform plan` — review the plan carefully; it should show on the order of 40-80 resources for a first-time VPC+EKS+node-group+IRSA apply.
4. `terraform apply` (expect 12-20 minutes for the EKS control plane to become active).
5. Configure `kubectl`:
   ```bash
   aws eks update-kubeconfig --name $(terraform output -raw cluster_name) --region us-east-1
   kubectl get nodes
   ```

**Success criteria:** `kubectl get nodes` shows your managed node group nodes as `Ready`, and `terraform output` exposes at minimum `cluster_endpoint`, `cluster_name`, and `oidc_provider_arn`.

---

### Lab 3 — IRSA role + aws-load-balancer-controller

1. Add the IRSA module for the load balancer controller (see 01-README's `lb_controller_irsa` block).
2. `terraform apply` to create the role, then install the controller via Helm, referencing the ServiceAccount annotation Terraform's IRSA module output gives you:
   ```bash
   helm repo add eks https://aws.github.io/eks-charts
   helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
     -n kube-system \
     --set clusterName=$(terraform output -raw cluster_name) \
     --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=$(terraform output -raw lb_controller_role_arn)
   ```
3. Verify: `kubectl get pods -n kube-system | grep aws-load-balancer-controller` shows `Running`.

**Success criteria:** The controller pod is running and its logs (`kubectl logs -n kube-system deploy/aws-load-balancer-controller`) show no permission errors when it starts reconciling.

---

### Lab 4 — Deploy a sample app with Helm via CI (the assigned hands-on activity, part 2)

1. Push your Terraform code to a GitHub repo. Add a minimal GitHub Actions workflow (see 03-README's example) that runs `terraform plan` on PRs against `environments/dev/**` and posts the plan as a PR comment.
2. Add a second, separate CD job (not a Terraform apply — a Kubernetes deploy step) that runs after the Terraform apply job succeeds, using `helm upgrade --install` to deploy a simple app (e.g., the standard `nginx` chart) with an `Ingress` targeting the ALB controller you just installed.
3. Open a PR changing a `.tf` file (e.g., bump `desired_size`), confirm the plan is posted as a comment, merge, and confirm the apply + subsequent Helm deploy both succeed via the Actions run log.

**Success criteria:** A merged PR triggers, in order, `terraform apply` then a Helm deployment, visible as a green CI run, and `kubectl get ingress` shows an ALB address that resolves to your sample app.

---

### Cleanup

```bash
helm uninstall nginx-sample
helm uninstall aws-load-balancer-controller -n kube-system
terraform -chdir=environments/dev destroy
aws dynamodb delete-table --table-name terraform-locks
aws s3 rb s3://my-tf-state-$(whoami) --force
```
Destroy order matters: uninstall anything that created AWS resources *outside* Terraform's knowledge (like ALBs created by the controller) before `terraform destroy`, or the VPC/security-group deletion can fail because those dangling resources still reference them.

### Stretch challenge

Split this single-environment project into `environments/dev` and `environments/prod`, each with its own backend `key` and `tfvars`, sharing a common local module (`modules/eks-cluster`) — then wire up Atlantis (via `atlantis.yaml` project blocks) instead of the GitHub Actions workflow, and compare the reviewer experience (plan-comment quality, locking behavior) between the two approaches firsthand.
