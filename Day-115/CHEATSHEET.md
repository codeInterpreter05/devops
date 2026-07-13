# Day 115 — Cheatsheet: Kubernetes Cost Optimisation

## OpenCost / Kubecost install & query

```bash
# Install OpenCost (needs Prometheus reachable in-cluster)
helm repo add opencost https://opencost.github.io/opencost-helm-chart
helm install opencost opencost/opencost -n opencost --create-namespace \
  --set opencost.prometheus.external.enabled=true \
  --set opencost.prometheus.external.url=http://<prometheus-svc>:9090

# Install Kubecost (commercial, free tier available)
helm repo add kubecost https://kubecost.github.io/cost-analyzer/
helm install kubecost kubecost/cost-analyzer -n kubecost --create-namespace

# Port-forward UI/API
kubectl port-forward -n opencost svc/opencost 9090:9090 9003:9003

# Allocation API — cost by namespace, last 7 days
curl "http://localhost:9003/allocation/compute?window=7d&aggregate=namespace" | jq

# Allocation API — by namespace + label
curl "http://localhost:9003/allocation/compute?window=7d&aggregate=namespace,label:team" | jq

# Allocation API — by controller (deployment/statefulset)
curl "http://localhost:9003/allocation/compute?window=7d&aggregate=controller" | jq

# Cluster-wide efficiency
curl "http://localhost:9003/allocation/compute?window=7d&aggregate=cluster" | jq
```

## Vertical Pod Autoscaler (VPA)

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Off"   # Off = recommend only | Initial = set on pod creation only | Auto = actively evict+resize
  resourcePolicy:
    containerPolicies:
      - containerName: '*'
        minAllowed: {cpu: 100m, memory: 128Mi}
        maxAllowed: {cpu: 2, memory: 4Gi}
```

```bash
kubectl apply -f vpa.yaml
kubectl describe vpa my-app-vpa           # see target / lowerBound / upperBound recommendations
kubectl get vpa my-app-vpa -o yaml        # raw recommendation data
```

## Manual right-sizing

```bash
kubectl set resources deployment/my-app --requests=cpu=250m,memory=256Mi --limits=cpu=500m,memory=512Mi
kubectl top pods -n my-namespace --containers    # live actual usage per container
kubectl describe node <node> | grep -A5 "Allocated resources"   # requested vs. capacity per node
```

## Karpenter

```yaml
# NodePool: allow Spot + On-Demand fallback, diversify instance types
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: spot-general
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]
        - key: node.kubernetes.io/instance-type
          operator: In
          values: ["m5.large","m5.xlarge","m6i.large","c5.large","c5.xlarge"]
      nodeClassRef: {group: karpenter.k8s.aws, kind: EC2NodeClass, name: default}
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 30s
  limits:
    cpu: "1000"
```

```bash
kubectl get nodepool                       # list NodePools
kubectl get nodeclaims                     # see nodes Karpenter has actually provisioned
kubectl describe nodeclaim <name>          # why a node was provisioned / its instance type
kubectl get nodes -L karpenter.sh/capacity-type   # see spot vs on-demand per node
```

## Taint critical workloads off Spot

```yaml
# NodePool
taints:
  - {key: dedicated, value: on-demand-critical, effect: NoSchedule}
---
# Pod spec
tolerations:
  - {key: dedicated, value: on-demand-critical, operator: Equal, effect: NoSchedule}
```

## Quick diagnostic commands

```bash
kubectl top nodes                                  # aggregate node CPU/mem usage
kubectl top pods -A --sort-by=cpu                   # highest CPU consumers cluster-wide
kubectl get pods -A -o json | jq '.items[] | select(.status.phase=="Running") |
  {name: .metadata.name, req: .spec.containers[0].resources.requests}'   # dump all requests
kubectl get poddisruptionbudget -A                  # check PDBs aren't blocking consolidation
```
