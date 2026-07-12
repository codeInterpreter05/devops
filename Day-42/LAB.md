# Day 42 — Lab: K8s + IaC + AWS Mock Interview

**Goal:** Run yourself (or a study partner) through a full timed mock-interview session covering the three exercises from this week's review: a 45-minute system design, a live Terraform state debugging scenario, and a K8s RBAC exercise — simulating real interview conditions, not an open-book review.

**Prerequisites:**
- A working Terraform project with an S3+DynamoDB (or equivalent) remote backend you can deliberately break.
- A Kubernetes cluster (`kind`/`minikube`) with `kubectl`.
- A timer. A blank whiteboard/notes app for the system design portion (no AI assistance, no searching — this is meant to simulate real interview pressure).
- Ideally, a study partner to play "interviewer" and probe your answers — if solo, record yourself narrating out loud and review the recording afterward.

---

### Lab 1 — The core hands-on activity: 45-minute timed system design

1. Set a 45-minute timer. Prompt (read it once, then no re-reading/searching):
   > "Design AWS infra for a 10k RPS API with auto-scaling, HA, and zero-downtime deploys."
2. Spend the first 5 minutes **only** asking/writing down clarifying questions (RPS shape, latency SLA, region scope, budget, team size) — do not draw anything yet.
3. Spend the next 15 minutes sketching the high-level architecture (Route 53 → CDN → ALB → EKS → data layer → observability/CI-CD) on paper or a whiteboard tool.
4. Spend the remaining 25 minutes deep-diving: pick auto-scaling (HPA + Karpenter/Cluster Autoscaler), the data layer trade-off (Aurora vs. DynamoDB, connection pooling), and zero-downtime deploy strategy (rolling vs. canary, expand/contract migrations) — narrate trade-offs out loud the whole time.
5. Afterward, self-grade (or have your partner grade) against this checklist: did you clarify requirements first? Did you address both pod-level and node-level scaling? Did you name at least 3 concrete trade-offs? Did you mention connection pooling/read replicas? Did you address schema migrations as part of "zero-downtime"?

**Success criteria:** You produced a complete, coherent architecture within 45 minutes and can point to at least 3 explicit trade-offs you named out loud, unprompted.

---

### Lab 2 — Terraform state debugging exercise

1. Set up a minimal Terraform project with a remote backend and 2-3 simple resources (e.g., an S3 bucket, a security group, an EC2 instance).
2. Apply it successfully once:
   ```bash
   terraform init && terraform apply -auto-approve
   ```
3. **Have a partner (or do it yourself and "forget" on purpose) introduce drift**: manually change a tag on one resource via the AWS console, and manually delete a second resource entirely via the console.
4. Without looking at what changed, run the full diagnostic sequence from memory:
   ```bash
   terraform plan
   terraform state list
   terraform state show <resource.address>
   terraform plan -refresh-only
   ```
5. Correctly identify: which resource has console-drift (tag change) vs. which was deleted entirely, and propose the right fix for each (`-refresh-only` apply for the drifted one to decide whether to accept or revert it; recreate or `terraform state rm` + re-apply for the deleted one).
6. Simulate a locked state (kill a `terraform apply` mid-run with Ctrl+C at the right moment, or manually create a lock entry in DynamoDB) and practice `terraform force-unlock` correctly, after first confirming (out loud) that no other apply is genuinely running.

**Success criteria:** You correctly diagnosed both drift scenarios using only `plan`/`state list`/`state show`/`-refresh-only`, without being told in advance what was changed.

---

### Lab 3 — K8s RBAC exercise

1. Create a namespace, a ServiceAccount, and a Role granting only `get`/`list` on `configmaps` (not pods, not secrets, not logs):
   ```bash
   kubectl create namespace rbac-lab
   kubectl create serviceaccount limited-sa -n rbac-lab
   kubectl create role configmap-reader --verb=get,list --resource=configmaps -n rbac-lab
   kubectl create rolebinding limited-sa-binding --role=configmap-reader --serviceaccount=rbac-lab:limited-sa -n rbac-lab
   ```
2. Verify exactly what it can and cannot do using `kubectl auth can-i`:
   ```bash
   kubectl auth can-i get configmaps --as=system:serviceaccount:rbac-lab:limited-sa -n rbac-lab
   kubectl auth can-i get pods --as=system:serviceaccount:rbac-lab:limited-sa -n rbac-lab
   kubectl auth can-i get pods/log --as=system:serviceaccount:rbac-lab:limited-sa -n rbac-lab
   kubectl auth can-i get configmaps --as=system:serviceaccount:rbac-lab:limited-sa -n default
   ```
3. Predict each answer *before* running the command, then compare — pay special attention to the `pods/log` subresource case and the cross-namespace case.
4. Extend the exercise: create a `ClusterRole` for `pod-reader` (get/list/watch on pods), bind it via a namespaced `RoleBinding` in `rbac-lab` only, and confirm it does **not** grant access in the `default` namespace.

**Success criteria:** You correctly predicted all four `can-i` results before running them, including explaining why `pods/log` access is denied despite having implicit expectations otherwise, and why the `ClusterRole`-via-`RoleBinding` pattern didn't leak into other namespaces.

---

### Cleanup

```bash
terraform destroy -auto-approve
kubectl delete namespace rbac-lab
```

### Stretch challenge

Combine all three: extend the system design from Lab 1 with a specific Terraform module structure for the EKS cluster (state debugging concerns: what goes in one state file vs. split into multiple), and a concrete RBAC plan for which teams/ServiceAccounts get which permissions on that cluster — presenting all three as one unified 60-minute mock interview answer.
