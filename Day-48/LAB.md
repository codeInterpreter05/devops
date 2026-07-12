# Day 48 — Lab: EKS Deep Dive

**Goal:** Replace Cluster Autoscaler with Karpenter on a test cluster and benchmark scale-up speed — the assigned hands-on activity — plus hands-on exposure to VPC CNI limits, Fargate profiles, and the new Access Entries model.

**Prerequisites:** An EKS cluster (reuse Day 47's), `kubectl`/`helm`/AWS CLI access with permissions to modify IAM (for Karpenter's IRSA role) and EC2. A cluster already running Cluster Autoscaler is ideal for the A/B comparison, but you can also start fresh with just Karpenter.

---

### Lab 1 — Baseline: Cluster Autoscaler scale-up timing

1. If not already running, install Cluster Autoscaler (via Helm) pointed at your existing managed node group's ASG, with `--balance-similar-node-groups` and correct IAM permissions via IRSA.
2. Trigger a scale-up by deploying a Deployment whose combined pod resource requests exceed current cluster capacity:
   ```bash
   kubectl create deployment scale-test --image=nginx:1.27 --replicas=1
   kubectl set resources deployment scale-test --requests=cpu=1500m,memory=2Gi
   kubectl scale deployment scale-test --replicas=20
   ```
3. Time from the first `Pending` pod appearing to all 20 becoming `Running`:
   ```bash
   kubectl get events --field-selector reason=TriggeredScaleUp -w
   kubectl get pods -l app=scale-test -w
   ```

**Success criteria:** A recorded wall-clock time (in seconds) from scale-up trigger to all pods `Running` under Cluster Autoscaler — your baseline.

---

### Lab 2 — Install Karpenter (the assigned hands-on activity, part 1)

1. Create Karpenter's IRSA role and instance profile (via the official Karpenter Terraform submodule or the `getting-started` script), then install via Helm:
   ```bash
   helm registry logout public.ecr.aws
   helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter \
     --namespace kube-system \
     --set settings.clusterName=$(terraform output -raw cluster_name) \
     --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=$(terraform output -raw karpenter_role_arn)
   ```
2. Apply a `NodePool` and `EC2NodeClass`:
   ```yaml
   apiVersion: karpenter.sh/v1
   kind: NodePool
   metadata: {name: default}
   spec:
     template:
       spec:
         requirements:
         - {key: karpenter.sh/capacity-type, operator: In, values: ["on-demand"]}
         nodeClassRef: {name: default, group: karpenter.k8s.aws, kind: EC2NodeClass}
     limits: {cpu: "100"}
     disruption: {consolidationPolicy: WhenEmptyOrUnderutilized, consolidateAfter: 30s}
   ```
3. Verify: `kubectl get nodepool` and `kubectl logs -n kube-system deploy/karpenter` show no errors.

**Success criteria:** Karpenter pod running healthy, `NodePool` and `EC2NodeClass` applied with no validation errors.

---

### Lab 3 — Scale down Cluster Autoscaler's node group, benchmark Karpenter (the assigned hands-on activity, part 2)

1. Set your existing managed node group's `minSize`/`desiredSize` to a low floor (don't delete it yet — safer migration) so new capacity is provisioned by Karpenter instead.
2. Delete and redeploy the scale-test Deployment to force fresh scheduling:
   ```bash
   kubectl delete deployment scale-test
   kubectl create deployment scale-test --image=nginx:1.27 --replicas=1
   kubectl set resources deployment scale-test --requests=cpu=1500m,memory=2Gi
   kubectl scale deployment scale-test --replicas=20
   ```
3. Time the same scale-up event under Karpenter, watching node provisioning directly:
   ```bash
   kubectl get nodeclaims -w
   kubectl get pods -l app=scale-test -w
   ```
4. Compare the two measured times and the actual instance types each system chose (`kubectl get nodes -L node.kubernetes.io/instance-type`).

**Success criteria:** A second recorded wall-clock time under Karpenter, and a clear side-by-side comparison against Lab 1's Cluster Autoscaler baseline — including which chose more appropriately-sized instances for the workload's actual resource requests.

---

### Lab 4 — VPC CNI pod density limit, observed directly

1. Pick a small instance type node (e.g., `t3.medium`) and check its max pod capacity: `kubectl describe node <node> | grep -A2 Allocatable`.
2. Deploy enough trivial pods (`kubectl scale deployment scale-test --replicas=<enough to exceed density limit> `, using minimal resource requests this time so CPU/memory isn't the limiting factor) to hit the ENI/IP-based pod density ceiling before CPU/memory would ever be exhausted.
3. Confirm via `kubectl describe pod <pending-pod>` that the scheduling failure reason references insufficient pod capacity on that node type, not CPU/memory.

**Success criteria:** You've directly observed a pod stuck `Pending` due to ENI/IP-based density limits rather than compute resource exhaustion, and can state the node's actual max-pods number from `kubectl describe node`.

---

### Cleanup

```bash
kubectl delete deployment scale-test
helm uninstall karpenter -n kube-system
kubectl delete nodepool default
kubectl delete ec2nodeclass default
helm uninstall cluster-autoscaler -n kube-system   # if installed for Lab 1
```

### Stretch challenge

Create an EKS Access Entry granting a second IAM role namespace-scoped `AmazonEKSEditPolicy` access (rather than cluster-admin), switch the cluster's authentication mode to `API_AND_CONFIG_MAP`, confirm both an existing `aws-auth`-mapped principal and your new Access-Entry-mapped principal can both authenticate, then fully cut over to `API` mode and confirm the `aws-auth` ConfigMap is no longer consulted.
