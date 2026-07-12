# Day 26 — Lab: RBAC & Security

**Goal:** Build a real least-privilege ServiceAccount from scratch, verify permissions with `kubectl auth can-i`, configure IRSA for real S3 access from an EKS pod, and see Pod Security Admission actually block an insecure pod.

**Prerequisites:** A running cluster for RBAC/PSA sections (minikube is fine). The IRSA section requires a real **EKS cluster** with `eksctl`/`aws` CLI configured and permissions to create IAM roles — note clearly where this diverges from the minikube-only sections.

---

### Lab 1 — The core hands-on activity (part 1): read-only ServiceAccount for a monitoring pod

1. Create a namespace, ServiceAccount, Role, and RoleBinding limited to read-only Pod access:
   ```bash
   kubectl create namespace monitoring
   kubectl create serviceaccount monitoring-reader -n monitoring
   cat <<'EOF' | kubectl apply -f -
   apiVersion: rbac.authorization.k8s.io/v1
   kind: Role
   metadata: { name: pod-reader, namespace: monitoring }
   rules:
     - apiGroups: [""]
       resources: ["pods", "pods/log"]
       verbs: ["get", "list", "watch"]
   EOF
   kubectl create rolebinding monitoring-reader-binding \
     --role=pod-reader --serviceaccount=monitoring:monitoring-reader -n monitoring
   ```
2. Verify what it CAN do:
   ```bash
   kubectl auth can-i list pods --as=system:serviceaccount:monitoring:monitoring-reader -n monitoring
   kubectl auth can-i get pods/log --as=system:serviceaccount:monitoring:monitoring-reader -n monitoring
   ```
3. Verify what it CANNOT do (should all return "no"):
   ```bash
   kubectl auth can-i delete pods --as=system:serviceaccount:monitoring:monitoring-reader -n monitoring
   kubectl auth can-i list secrets --as=system:serviceaccount:monitoring:monitoring-reader -n monitoring
   kubectl auth can-i list pods --as=system:serviceaccount:monitoring:monitoring-reader -n default
   ```
4. Prove it in a real pod, not just via `--as` impersonation:
   ```bash
   kubectl run test-agent --image=bitnami/kubectl:latest -n monitoring \
     --overrides='{"spec":{"serviceAccountName":"monitoring-reader"}}' \
     --command -- sleep 3600
   kubectl exec -n monitoring test-agent -- kubectl get pods -n monitoring    # works
   kubectl exec -n monitoring test-agent -- kubectl delete pod test-agent -n monitoring  # should be Forbidden
   ```

**Success criteria:** You've confirmed, both via `--as` impersonation and via a real running pod, that the ServiceAccount can list/read pods but cannot delete pods, read secrets, or act outside its own namespace.

---

### Lab 2 — The core hands-on activity (part 2): IRSA for S3 access (EKS only)

Skip to Lab 3 if you don't have an EKS cluster available — read through it carefully anyway since this is the assigned interview topic.

1. Ensure the cluster has an IAM OIDC provider associated:
   ```bash
   eksctl utils associate-iam-oidc-provider --cluster <your-cluster> --approve
   ```
2. Create an IAM policy scoped to one S3 bucket, and a role trusting only a specific ServiceAccount:
   ```bash
   eksctl create iamserviceaccount \
     --cluster <your-cluster> \
     --namespace prod \
     --name s3-uploader \
     --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
     --approve
   ```
   (In a real scenario you'd scope this to a custom least-privilege policy for one bucket, not the broad managed policy — the managed policy is used here only to keep the lab short.)
3. Confirm the ServiceAccount annotation was created automatically:
   ```bash
   kubectl get sa s3-uploader -n prod -o yaml | grep role-arn
   ```
4. Run a pod using that ServiceAccount and confirm real AWS access with no static credentials anywhere:
   ```bash
   kubectl run s3-test --image=amazon/aws-cli -n prod \
     --overrides='{"spec":{"serviceAccountName":"s3-uploader"}}' \
     --command -- sleep 3600
   kubectl exec -n prod s3-test -- env | grep AWS_
   kubectl exec -n prod s3-test -- aws sts get-caller-identity
   kubectl exec -n prod s3-test -- aws s3 ls
   ```

**Success criteria:** You've confirmed `aws sts get-caller-identity` inside the pod shows an **assumed role** session (not an IAM user), and that no `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` environment variables or Secrets exist anywhere in the pod's configuration.

---

### Lab 3 — Pod Security Admission in action

1. Enforce the `restricted` profile on a namespace:
   ```bash
   kubectl create namespace secure-ns
   kubectl label namespace secure-ns pod-security.kubernetes.io/enforce=restricted
   ```
2. Attempt to run a privileged pod and watch it get rejected:
   ```bash
   kubectl run bad-pod --image=nginx -n secure-ns --overrides='{"spec":{"containers":[{"name":"nginx","image":"nginx","securityContext":{"privileged":true}}]}}'
   ```
3. Fix it to comply with `restricted` and confirm it's accepted:
   ```bash
   cat <<'EOF' | kubectl apply -f -
   apiVersion: v1
   kind: Pod
   metadata: { name: good-pod, namespace: secure-ns }
   spec:
     securityContext:
       runAsNonRoot: true
       runAsUser: 10001
       seccompProfile: { type: RuntimeDefault }
     containers:
       - name: app
         image: nginxinc/nginx-unprivileged:1.25
         securityContext:
           allowPrivilegeEscalation: false
           capabilities: { drop: ["ALL"] }
   EOF
   kubectl get pod good-pod -n secure-ns
   ```

**Success criteria:** The privileged pod is rejected outright with a Pod Security Admission error message, and the corrected, `restricted`-compliant pod starts successfully.

---

### Cleanup

```bash
kubectl delete namespace monitoring secure-ns --ignore-not-found
kubectl delete namespace prod --ignore-not-found   # only if created solely for this lab
eksctl delete iamserviceaccount --cluster <your-cluster> --namespace prod --name s3-uploader   # EKS only
```

### Stretch challenge

Write an OPA Gatekeeper (or Kyverno) `ConstraintTemplate`/policy that rejects any Pod without CPU and memory limits set on every container, apply it to `secure-ns`, and confirm both that a non-compliant pod is rejected and that the earlier `good-pod` (once you add limits to it) passes.
