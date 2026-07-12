# Day 45 — Lab: K8s Advanced Scheduling

**Goal:** Configure real scheduling constraints on a multi-node/multi-AZ cluster — spread replicas across AZs, and structurally guarantee critical pods can never land on spot/GPU-tainted nodes.

**Prerequisites:** A Kubernetes cluster with nodes labeled by zone (`topology.kubernetes.io/zone` — true by default on EKS/GKE/AKS; if using kind/minikube, you'll need to manually label nodes to simulate zones), `kubectl` access with permission to taint nodes and create PriorityClasses.

---

### Lab 1 — Label/verify zone topology

1. Check what zone labels your nodes already have:
   ```bash
   kubectl get nodes -L topology.kubernetes.io/zone
   ```
2. If using a local multi-node cluster (kind/minikube) without real zones, simulate them:
   ```bash
   kubectl label node worker-1 topology.kubernetes.io/zone=zone-a
   kubectl label node worker-2 topology.kubernetes.io/zone=zone-b
   kubectl label node worker-3 topology.kubernetes.io/zone=zone-c
   ```

**Success criteria:** `kubectl get nodes -L topology.kubernetes.io/zone` shows at least 3 distinct zone values across your nodes.

---

### Lab 2 — Configure topology spread to distribute replicas across AZs (the assigned hands-on activity, part 1)

1. Deploy a 6-replica Deployment with a topology spread constraint:
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: web-frontend
   spec:
     replicas: 6
     selector:
       matchLabels: {app: web-frontend}
     template:
       metadata:
         labels: {app: web-frontend}
       spec:
         topologySpreadConstraints:
         - maxSkew: 1
           topologyKey: topology.kubernetes.io/zone
           whenUnsatisfiable: DoNotSchedule
           labelSelector:
             matchLabels: {app: web-frontend}
         containers:
         - name: web
           image: nginx:1.27
           resources:
             requests: {cpu: "50m", memory: "64Mi"}
   ```
2. Apply and check distribution:
   ```bash
   kubectl apply -f web-frontend.yaml
   kubectl get pods -l app=web-frontend -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName
   kubectl get pods -l app=web-frontend -o wide | awk '{print $7}' | sort | uniq -c   # rough node distribution
   ```
3. Cross-reference each node to its zone label to confirm even spread (2 replicas per zone for 6 replicas / 3 zones).
4. Now try the same with `podAntiAffinity` (`requiredDuringSchedulingIgnoredDuringExecution`, `topologyKey: topology.kubernetes.io/zone`) instead, scale to 4 replicas across 3 zones, and observe what happens to the 4th replica — compare the failure mode to the topology spread approach.

**Success criteria:** You can show, with `kubectl` output, that replicas are evenly spread across zones, and you can explain in one sentence why topology spread constraints handled an uneven replica-to-zone ratio more gracefully than strict pod anti-affinity did.

---

### Lab 3 — Taint spot nodes, add GPU toleration (the assigned hands-on activity, part 2)

1. Simulate a spot node pool by tainting one or more nodes:
   ```bash
   kubectl taint nodes worker-2 lifecycle=spot:NoSchedule
   kubectl label node worker-2 lifecycle=spot
   ```
2. Deploy a "critical" pod with **no toleration** and confirm it never lands on the tainted node even under pressure:
   ```bash
   kubectl run critical-app --image=nginx:1.27 --overrides='{"spec":{"priorityClassName":"business-critical"}}'
   kubectl get pod critical-app -o wide
   ```
   (Create the `business-critical` PriorityClass first — see Lab 4.)
3. Deploy a batch/spot-tolerant pod **with** a matching toleration and confirm it can (but isn't forced to) land on the tainted node:
   ```yaml
   tolerations:
   - key: "lifecycle"
     operator: "Equal"
     value: "spot"
     effect: "NoSchedule"
   nodeSelector:
     lifecycle: spot
   ```
4. Additionally simulate a GPU node needing an explicit opt-in taint:
   ```bash
   kubectl taint nodes worker-3 nvidia.com/gpu=present:NoSchedule
   ```
   and confirm only pods with a matching toleration (representing GPU workloads) can be scheduled there.

**Success criteria:** You can demonstrate that a pod without the spot toleration is structurally unable to land on the tainted node (not just "unlikely to," but impossible), and articulate why this is stronger than relying on node affinity alone.

---

### Lab 4 — PriorityClass and preemption in action

1. Create two PriorityClasses:
   ```bash
   kubectl apply -f - <<EOF
   apiVersion: scheduling.k8s.io/v1
   kind: PriorityClass
   metadata: {name: business-critical}
   value: 1000000
   globalDefault: false
   description: "critical workloads"
   ---
   apiVersion: scheduling.k8s.io/v1
   kind: PriorityClass
   metadata: {name: batch-low}
   value: 100
   globalDefault: true
   description: "default for unspecified workloads"
   EOF
   ```
2. Fill a node close to capacity with several `batch-low` pods (set explicit resource requests that sum near the node's allocatable capacity).
3. Deploy one `business-critical` pod requesting enough resources that it can only fit by evicting a `batch-low` pod. Watch:
   ```bash
   kubectl get events --sort-by=.lastTimestamp | grep -i preempt
   ```

**Success criteria:** You observe a `Preempted` event and confirm a lower-priority pod was evicted to make room for the higher-priority one.

---

### Cleanup

```bash
kubectl delete deployment web-frontend
kubectl delete pod critical-app
kubectl taint nodes worker-2 lifecycle=spot:NoSchedule-
kubectl taint nodes worker-3 nvidia.com/gpu=present:NoSchedule-
kubectl delete priorityclass business-critical batch-low
```

### Stretch challenge

Install the [descheduler](https://github.com/kubernetes-sigs/descheduler) via its Helm chart, configure the `LowNodeUtilization` and `RemovePodsViolatingTopologySpreadConstraint` strategies, then deliberately unbalance your cluster (cordon/drain one zone's node, let pods pile onto another, then uncordon) and observe whether the descheduler rebalances the pods back toward even zone distribution on its next run.
