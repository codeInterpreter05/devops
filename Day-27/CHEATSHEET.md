# Day 27 — Cheatsheet: Autoscaling

## HPA

```bash
kubectl autoscale deployment web --cpu-percent=70 --min=2 --max=20
kubectl get hpa
kubectl describe hpa web
kubectl top pods                 # requires metrics-server
kubectl top nodes
```

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: web-hpa }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: web }
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 70 } }
  behavior:
    scaleUp:   { stabilizationWindowSeconds: 0,   policies: [{ type: Pods, value: 4, periodSeconds: 60 }] }
    scaleDown: { stabilizationWindowSeconds: 300, policies: [{ type: Pods, value: 1, periodSeconds: 60 }] }
```

CPU % is computed against `resources.requests.cpu` — no request set means `<unknown>`, no scaling.

## VPA

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata: { name: web-vpa }
spec:
  targetRef: { apiVersion: apps/v1, kind: Deployment, name: web }
  updatePolicy: { updateMode: "Off" }   # Off (recommend-only) | Initial | Recreate | Auto
```

```bash
kubectl describe vpa web-vpa    # target / lowerBound / upperBound recommendations
```

## KEDA

```bash
helm install keda kedacore/keda -n keda --create-namespace
kubectl get scaledobject
kubectl describe scaledobject <name>
kubectl get hpa                 # KEDA auto-creates: keda-hpa-<scaledobject-name>
```

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata: { name: consumer-scaler }
spec:
  scaleTargetRef: { name: consumer-deployment }
  minReplicaCount: 0             # KEDA-only superpower: true scale-to-zero
  maxReplicaCount: 30
  pollingInterval: 15
  cooldownPeriod: 300
  triggers:
    - type: aws-sqs-queue        # or: rabbitmq, kafka, prometheus, cron, postgresql, ...
      metadata: { queueURL: "...", queueLength: "5", awsRegion: us-east-1 }
      authenticationRef: { name: keda-trigger-auth }
```

## Cluster Autoscaler (ASG-based)

```bash
kubectl get pods -n kube-system -l app=cluster-autoscaler
kubectl logs -n kube-system -l app=cluster-autoscaler | grep -i scale-up
kubectl describe configmap cluster-autoscaler-status -n kube-system
```

Key flags: `--nodes=<min>:<max>:<asg-name>`, `--scale-down-unneeded-time=10m`

## Karpenter (direct EC2 provisioning, no node groups)

```bash
kubectl get nodepool
kubectl get nodeclaims
kubectl get ec2nodeclass
kubectl logs -n karpenter -l app.kubernetes.io/name=karpenter
```

```yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata: { name: general-purpose }
spec:
  template:
    spec:
      requirements:
        - { key: karpenter.sh/capacity-type, operator: In, values: ["spot", "on-demand"] }
      nodeClassRef: { name: default }
  disruption:
    consolidationPolicy: WhenUnderutilized
    consolidateAfter: 30s
```

## Diagnosing "pods stuck Pending"

```bash
kubectl describe pod <pod> | grep -A10 Events   # FailedScheduling / Insufficient cpu|memory
kubectl get events --field-selector reason=FailedScheduling
```

## Cooldown / flapping-prevention cheat

```
HPA:                behavior.scaleDown.stabilizationWindowSeconds (default 300s)
KEDA:                cooldownPeriod
Cluster Autoscaler:  --scale-down-unneeded-time (default 10m)
Karpenter:            disruption.consolidateAfter
Rule of thumb: scale up FAST, scale down SLOW.
```
