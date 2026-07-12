# Day 53 — Cheatsheet: Phase 1 Project — Full Stack IaC

## Architecture layer order

```
VPC (3 AZs, public+private subnets, NAT per AZ)
  -> EKS (control plane + managed node groups, in private subnets)
       -> IRSA roles (Load Balancer Controller, ESO, Karpenter, EBS CSI)
  -> RDS (Multi-AZ, manage_master_user_password=true) + ElastiCache (replication group)
  -> ALB (provisioned by in-cluster Load Balancer Controller reacting to Ingress)
```

## Terraform — staged apply

```bash
terraform init
terraform plan -target=module.vpc
terraform apply -target=module.vpc
terraform apply -target=module.eks
terraform apply                       # remaining resources
terraform state list
terraform destroy                     # full teardown
```

## EKS + kubeconfig

```bash
aws eks update-cluster-version --name prod-eks-cluster --kubernetes-version 1.29
aws eks update-kubeconfig --name prod-eks-cluster --region us-east-1
kubectl get nodes
```

## Helm

```bash
helm install my-app ./my-app -f values-prod.yaml -n production
helm upgrade my-app ./my-app -f values-prod.yaml -n production
helm rollback my-app 1 -n production
helm history my-app -n production
```

## ArgoCD

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
argocd login localhost:8080
argocd app create my-app --repo <REPO_URL> --path charts/my-app --dest-server https://kubernetes.default.svc --dest-namespace production
argocd app sync my-app
argocd app get my-app
```
```yaml
syncPolicy:
  automated: { prune: true, selfHeal: true }
```

## External Secrets Operator

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
spec:
  provider:
    aws: { service: SecretsManager, region: us-east-1, auth: { jwt: { serviceAccountRef: { name: external-secrets-sa } } } }
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
spec:
  secretStoreRef: { name: aws-secrets-manager, kind: SecretStore }
  target: { name: app-db-credentials }
  refreshInterval: 1h
  data:
    - { secretKey: password, remoteRef: { key: prod/app/db, property: password } }
```

## KEDA

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
spec:
  scaleTargetRef: { name: order-worker }
  minReplicaCount: 0
  maxReplicaCount: 20
  cooldownPeriod: 300
  triggers:
    - type: aws-sqs-queue
      metadata: { queueURL: "<URL>", queueLength: "5" }
      authenticationRef: { name: keda-aws-credentials }
```
```bash
kubectl get scaledobject -n production
kubectl get hpa -n production          # KEDA manages this under the hood
```

## Karpenter

```yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
spec:
  template:
    spec:
      requirements:
        - { key: karpenter.sh/capacity-type, operator: In, values: ["spot", "on-demand"] }
      nodeClassRef: { name: default }
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    expireAfter: 720h
```
```bash
kubectl get nodepool
kubectl get nodeclaim
kubectl logs -n karpenter deployment/karpenter -f    # first place to check if nodes aren't provisioning
```

## Full elastic chain (memorize for interview)

```
event (SQS depth) -> KEDA scales pods (0 -> N) -> pods Pending (no capacity)
  -> Karpenter provisions matching nodes -> pods schedule -> queue drains
  -> KEDA scales pods back down -> Karpenter consolidates/terminates idle nodes
```
