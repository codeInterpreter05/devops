# Day 42 — Cheatsheet: K8s + IaC + AWS Mock Interview

## System design answer skeleton (45-min timing)

```
0-5 min    clarify: RPS shape, latency SLA, region scope, consistency, budget
5-20 min   high-level: Route53 -> CDN -> ALB -> EKS -> data layer -> observability/CI-CD
20-45 min  deep-dive 2-3 areas + name trade-offs out loud:
             - compute/scaling (HPA + Karpenter/Cluster Autoscaler)
             - data layer (Aurora vs DynamoDB, connection pooling, read replicas)
             - deploy strategy (rolling vs canary/blue-green, expand/contract migrations)
```

## EKS high-traffic API building blocks

```
Route 53          latency/geo routing + health checks
CloudFront + Shield  CDN, edge cache, DDoS absorption
ALB / AWS LB Controller   L7 load balancing into EKS
EKS + Karpenter/Cluster Autoscaler   node-level scaling
HPA (custom metrics via Prometheus Adapter)   pod-level scaling
Aurora (Multi-AZ + read replicas) / DynamoDB   data layer
RDS Proxy / PgBouncer   connection pooling — bottleneck at high RPS
ElastiCache (Redis)   caching layer
Secrets Manager/Vault + ESO + IRSA   secrets injection
Argo Rollouts / Istio   canary, blue-green
ArgoCD/Flux   GitOps delivery
```

## Terraform state debugging sequence

```bash
terraform plan                       # step 1, always
terraform state list                 # what Terraform believes it manages
terraform state show <addr>           # full recorded attributes for one resource
terraform plan -refresh-only          # reconcile state to reality, no infra change
terraform apply -refresh-only         # commit that reconciliation to state

terraform import <addr> <cloud-id>    # bring an existing real resource under management
terraform state rm <addr>              # drop a stale state entry (real infra untouched)
terraform state mv <src> <dst>         # rename/relocate in state (refactor-safe)
terraform force-unlock <LOCK_ID>       # only after confirming no other apply is running

terraform state pull > backup.tfstate  # ALWAYS back up before manual state surgery
terraform state push backup.tfstate
```

## Force-new vs. in-place: reading a plan

```
~  update in-place       (no replacement)
-/+  destroy and re-create  (force-new attribute changed)
+    create
-    destroy
```

## Kubernetes RBAC quick reference

```bash
kubectl create role <name> --verb=get,list --resource=configmaps -n <ns>
kubectl create clusterrole <name> --verb=get,list,watch --resource=pods
kubectl create rolebinding <name> --role=<role> --serviceaccount=<ns>:<sa> -n <ns>
kubectl create clusterrolebinding <name> --clusterrole=<role> --serviceaccount=<ns>:<sa>

kubectl auth can-i get pods --as=system:serviceaccount:<ns>:<sa> -n <ns>
kubectl auth can-i '*' '*' --as=<user>          # check admin-equivalent access
kubectl get rolebindings,clusterrolebindings -A -o wide
```

```yaml
# subresources need explicit entries — get on pods != get on pods/log
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log", "pods/exec"]
    verbs: ["get", "list", "watch"]
```

## Scope cheat table

```
Role              namespaced permission set
ClusterRole       cluster-scoped permission set (reusable)
RoleBinding       grants Role OR ClusterRole, SCOPED to one namespace
ClusterRoleBinding grants ClusterRole CLUSTER-WIDE
```
