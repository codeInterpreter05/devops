# Day 48 — Cheatsheet: EKS Deep Dive

## Managed Node Groups vs Fargate

```bash
# Managed node group
aws eks create-nodegroup --cluster-name prod --nodegroup-name ng-1 \
  --node-role arn:aws:iam::123456789012:role/eks-node-role \
  --subnets subnet-abc subnet-def --instance-types m6i.large \
  --scaling-config minSize=2,maxSize=6,desiredSize=3

# Fargate profile
aws eks create-fargate-profile --cluster-name prod --fargate-profile-name fp-batch \
  --pod-execution-role-arn arn:aws:iam::123456789012:role/fargate-pod-exec \
  --subnets subnet-abc subnet-def \
  --selectors namespace=batch-jobs
```
Fargate: no DaemonSets, no hostNetwork/privileged, no GPU, per-pod billing, slower cold start.

## EKS Add-ons

```bash
aws eks create-addon --cluster-name prod --addon-name vpc-cni
aws eks create-addon --cluster-name prod --addon-name coredns
aws eks create-addon --cluster-name prod --addon-name kube-proxy
aws eks describe-addon-versions --addon-name vpc-cni --kubernetes-version 1.30
aws eks update-addon --cluster-name prod --addon-name vpc-cni --addon-version v1.18.0-eksbuild.1
```

## VPC CNI internals

```bash
kubectl get ds aws-node -n kube-system              # VPC CNI daemonset
kubectl describe node <node> | grep -A2 Allocatable  # max pods for this instance type

# Enable prefix delegation (16x pod density)
kubectl set env daemonset aws-node -n kube-system ENABLE_PREFIX_DELEGATION=true

# Enable Security Groups for Pods
kubectl set env daemonset aws-node -n kube-system ENABLE_POD_ENI=true
```
CNI model: pods get real routable VPC IPs (ENIs + secondary IPs), not an overlay/virtual network like Calico/Flannel.
ALB IP-target mode → routes directly to pod IPs, skipping kube-proxy/NodePort hop.

## Cluster Autoscaler vs Karpenter

```
Cluster Autoscaler: scales pre-defined ASGs/node groups up/down only. No instance-type choice.
Karpenter: calls EC2 RunInstances/CreateFleet directly. Picks best-fit instance type/AZ/capacity-type per pending pod batch. Also consolidates (repacks/removes underutilized nodes).
```

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata: {name: default}
spec:
  template:
    spec:
      requirements:
      - {key: karpenter.sh/capacity-type, operator: In, values: ["spot","on-demand"]}
      nodeClassRef: {name: default, group: karpenter.k8s.aws, kind: EC2NodeClass}
  limits: {cpu: "1000"}
  disruption: {consolidationPolicy: WhenEmptyOrUnderutilized, consolidateAfter: 30s}
```

```bash
kubectl get nodepool
kubectl get nodeclaims
kubectl logs -n kube-system deploy/karpenter
```

## EKS Distro / EKS Anywhere

```bash
eksctl anywhere create cluster -f cluster-spec.yaml   # vSphere/bare-metal/Docker/Snow/CloudStack
eksctl anywhere upgrade cluster -f cluster-spec.yaml
```
EKS-D = open-source Kubernetes build AWS also uses internally. EKS-A = tooling to run it on your own infra. Neither = AWS-managed control plane.

## Access Entries (new RBAC model)

```bash
aws eks update-cluster-config --name prod --access-config authenticationMode=API_AND_CONFIG_MAP
aws eks create-access-entry --cluster-name prod --principal-arn arn:aws:iam::123456789012:role/team-a
aws eks associate-access-policy --cluster-name prod \
  --principal-arn arn:aws:iam::123456789012:role/team-a \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSEditPolicy \
  --access-scope type=namespace,namespaces=team-a-ns
aws eks list-access-entries --cluster-name prod
aws eks update-cluster-config --name prod --access-config authenticationMode=API   # full cutover
```
Managed policies: `AmazonEKSClusterAdminPolicy`, `AmazonEKSAdminPolicy`, `AmazonEKSEditPolicy`, `AmazonEKSViewPolicy`.
