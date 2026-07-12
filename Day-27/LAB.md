# Day 27 — Lab: Autoscaling

**Goal:** Configure and trigger real HPA scaling under CPU load, deploy KEDA and scale a consumer based on RabbitMQ queue depth (the assigned hands-on activity), and understand cluster/node-level autoscaling conceptually via Karpenter's dry-run tooling.

**Prerequisites:** minikube (with `metrics-server` addon) for Labs 1-2; `helm` for KEDA and RabbitMQ. Lab 3's node-autoscaling concepts are best practiced on a real EKS cluster, noted inline.

```bash
minikube start --driver=docker --cpus=4 --memory=6g
minikube addons enable metrics-server
kubectl get deployment metrics-server -n kube-system -w   # wait for Available
```

---

### Lab 1 — HPA on CPU, triggered by real load

1. Deploy a CPU-bound test app with an explicit CPU request (required for HPA to compute a percentage):
   ```bash
   kubectl create deployment cpu-app --image=vish/stress --replicas=1 -- -cpus 1
   kubectl set resources deployment/cpu-app --requests=cpu=100m --limits=cpu=200m
   kubectl autoscale deployment cpu-app --cpu-percent=50 --min=1 --max=5
   kubectl get hpa -w
   ```
2. In a separate terminal, watch it scale up as the stress container consumes CPU:
   ```bash
   kubectl top pods -l app=cpu-app
   kubectl describe hpa cpu-app
   ```
3. Remove the load (scale the stress args down or delete/recreate with less CPU usage) and watch it scale back down — note how much slower scale-down is than scale-up (`stabilizationWindowSeconds`):
   ```bash
   kubectl get hpa cpu-app -w
   ```
4. Deliberately remove the CPU request and recreate the HPA to see the failure mode:
   ```bash
   kubectl set resources deployment/cpu-app --requests=cpu-   # remove the request
   kubectl describe hpa cpu-app | grep -A3 "current"           # should show <unknown>
   ```

**Success criteria:** You've watched real scale-up happen under CPU load, observed scale-down taking noticeably longer (by design), and reproduced the `<unknown>` metric failure mode caused by a missing CPU request.

---

### Lab 2 — The core hands-on activity: KEDA scaling on RabbitMQ queue depth

This is the assigned hands-on activity — do it for real, reusing any RabbitMQ familiarity from prior experience.

1. Install KEDA and RabbitMQ:
   ```bash
   helm repo add kedacore https://kedacore.github.io/charts
   helm repo add bitnami https://charts.bitnami.com/bitnami
   helm repo update
   helm install keda kedacore/keda --namespace keda --create-namespace
   helm install rabbitmq bitnami/rabbitmq --set auth.username=guest --set auth.password=guest
   kubectl get pods -n keda -w
   ```
2. Deploy a simple consumer Deployment (starts at 1 replica, scaled by KEDA) and a `ScaledObject`:
   ```bash
   kubectl create deployment queue-consumer --image=busybox:1.36 -- sh -c "sleep 3600"
   cat <<'EOF' | kubectl apply -f -
   apiVersion: keda.sh/v1alpha1
   kind: ScaledObject
   metadata: { name: queue-consumer-scaler }
   spec:
     scaleTargetRef: { name: queue-consumer }
     minReplicaCount: 1
     maxReplicaCount: 10
     cooldownPeriod: 60
     triggers:
       - type: rabbitmq
         metadata:
           queueName: work-queue
           mode: QueueLength
           value: "5"
           host: amqp://guest:guest@rabbitmq.default.svc.cluster.local:5672
   EOF
   kubectl get scaledobject queue-consumer-scaler
   kubectl get hpa    # confirm KEDA auto-created an HPA object named keda-hpa-queue-consumer-scaler
   ```
3. Push messages into the queue to simulate a backlog and watch it scale up:
   ```bash
   kubectl run rabbitmq-publisher --rm -it --image=pivotalrabbitmq/perf-test --restart=Never -- \
     --uri amqp://guest:guest@rabbitmq.default.svc.cluster.local:5672 \
     --queue work-queue --producers 1 --consumers 0 --rate 50 --time 30
   kubectl get pods -l app=queue-consumer -w      # should scale up as depth rises
   kubectl describe scaledobject queue-consumer-scaler
   ```
4. Stop publishing and watch it scale back down after `cooldownPeriod`:
   ```bash
   kubectl get deployment queue-consumer -w
   ```

**Success criteria:** You've confirmed KEDA created a real HPA under the hood, watched replica count rise in response to actual queue depth (not CPU), and watched it scale back down to `minReplicaCount` after the cooldown period once the queue drained.

---

### Lab 3 — Understanding node-level autoscaling (conceptual + EKS if available)

1. On minikube, simulate an unschedulable pod to see the pod-level symptom node autoscalers react to:
   ```bash
   kubectl run huge-pod --image=nginx --requests='cpu=8000m'
   kubectl describe pod huge-pod | grep -A5 Events    # Insufficient cpu -- this is exactly what CA/Karpenter watch for
   kubectl delete pod huge-pod
   ```
2. If you have access to an EKS cluster with Karpenter installed, inspect its decision-making directly:
   ```bash
   kubectl get nodepool
   kubectl get nodeclaims
   kubectl logs -n karpenter -l app.kubernetes.io/name=karpenter --tail=100
   ```
3. If you have Cluster Autoscaler installed instead, compare:
   ```bash
   kubectl get configmap cluster-autoscaler-status -n kube-system -o yaml
   ```

**Success criteria:** You can point to a concrete `FailedScheduling`/`Insufficient cpu` event and explain, in your own words, that this exact signal is what triggers a node autoscaler (CA or Karpenter) to provision new capacity — distinct from what triggers HPA/KEDA to request more pods in the first place.

---

### Cleanup

```bash
kubectl delete hpa cpu-app --ignore-not-found
kubectl delete deployment cpu-app queue-consumer --ignore-not-found
kubectl delete scaledobject queue-consumer-scaler --ignore-not-found
helm uninstall rabbitmq keda -n default 2>/dev/null; helm uninstall keda -n keda 2>/dev/null
kubectl delete namespace keda --ignore-not-found
minikube stop
```

### Stretch challenge

Reconfigure the KEDA `ScaledObject` with `minReplicaCount: 0` and a short `cooldownPeriod`, then measure (with `time` around a publish-then-consume cycle) exactly how long it takes from "first message published to an empty queue with 0 consumers" to "first consumer pod is Running and Ready." This is the real cold-start cost of scale-to-zero — explain when that tradeoff is and isn't acceptable for a production workload.
