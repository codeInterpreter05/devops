# Day 50 — Lab: Probes, Resources & QoS

**Goal:** Tune resource requests/limits and probes on a real app, observe the resulting QoS class and scheduling behavior, and deliberately trigger + diagnose an OOMKill.

**Prerequisites:** A running Kubernetes cluster (minikube, kind, or a real cluster) with `kubectl` configured, and `metrics-server` installed (`minikube addons enable metrics-server` or `kubectl apply` the metrics-server manifest) so `kubectl top` works.

---

### Lab 1 — Observe QoS classes directly

1. Create three pods with different resource configurations:
   ```bash
   kubectl create namespace qos-lab

   cat <<'EOF' | kubectl apply -f -
   apiVersion: v1
   kind: Pod
   metadata:
     name: guaranteed-pod
     namespace: qos-lab
   spec:
     containers:
       - name: app
         image: nginx
         resources:
           requests: { cpu: "100m", memory: "128Mi" }
           limits:   { cpu: "100m", memory: "128Mi" }
   ---
   apiVersion: v1
   kind: Pod
   metadata:
     name: burstable-pod
     namespace: qos-lab
   spec:
     containers:
       - name: app
         image: nginx
         resources:
           requests: { cpu: "100m", memory: "128Mi" }
           limits:   { cpu: "500m", memory: "256Mi" }
   ---
   apiVersion: v1
   kind: Pod
   metadata:
     name: besteffort-pod
     namespace: qos-lab
   spec:
     containers:
       - name: app
         image: nginx
   EOF
   ```
2. Check each pod's derived QoS class:
   ```bash
   for p in guaranteed-pod burstable-pod besteffort-pod; do
     echo "$p: $(kubectl get pod $p -n qos-lab -o jsonpath='{.status.qosClass}')"
   done
   ```

**Success criteria:** Output shows `Guaranteed`, `Burstable`, `BestEffort` matching the pods you'd predict from the YAML alone, before running the command.

---

### Lab 2 — Trigger and diagnose a real OOMKill

1. Deploy a container deliberately given too little memory for a workload that consumes more:
   ```bash
   cat <<'EOF' | kubectl apply -f -
   apiVersion: v1
   kind: Pod
   metadata:
     name: oom-demo
     namespace: qos-lab
   spec:
     containers:
       - name: stress
         image: polinux/stress
         resources:
           requests: { memory: "50Mi" }
           limits:   { memory: "50Mi" }
         command: ["stress"]
         args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
   EOF
   ```
2. Watch it get killed:
   ```bash
   kubectl get pod oom-demo -n qos-lab -w
   ```
3. Confirm the exact failure mode:
   ```bash
   kubectl describe pod oom-demo -n qos-lab | grep -A5 "Last State"
   ```
   Confirm you see `Reason: OOMKilled` and `Exit Code: 137`.

**Success criteria:** You can point to the exact fields in `kubectl describe pod` output that confirm an OOMKill versus any other failure reason, and explain in your own words why exit code 137 specifically means SIGKILL.

---

### Lab 3 — Probes: readiness vs. liveness in action (core hands-on activity)

1. Deploy an app with a toggle-able health endpoint (using `httpbin` for easy manual control isn't quite right — instead use a simple pattern with `busybox` and a flag file):
   ```bash
   cat <<'EOF' | kubectl apply -f -
   apiVersion: v1
   kind: Pod
   metadata:
     name: probe-demo
     namespace: qos-lab
   spec:
     containers:
       - name: app
         image: busybox
         command: ["sh", "-c", "touch /tmp/healthy; touch /tmp/ready; sleep 3600"]
         readinessProbe:
           exec:
             command: ["cat", "/tmp/ready"]
           periodSeconds: 5
         livenessProbe:
           exec:
             command: ["cat", "/tmp/healthy"]
           periodSeconds: 5
           failureThreshold: 2
   EOF
   ```
2. Confirm the pod is `Running` and `Ready`:
   ```bash
   kubectl get pod probe-demo -n qos-lab
   ```
3. Simulate a "not ready but still alive" state — remove only the readiness file:
   ```bash
   kubectl exec probe-demo -n qos-lab -- rm /tmp/ready
   kubectl get pod probe-demo -n qos-lab -w   # watch READY flip to 0/1, container does NOT restart
   ```
   Restore it: `kubectl exec probe-demo -n qos-lab -- touch /tmp/ready`
4. Now simulate a liveness failure — remove the health file:
   ```bash
   kubectl exec probe-demo -n qos-lab -- rm /tmp/healthy
   kubectl get pod probe-demo -n qos-lab -w   # watch RESTARTS count increment after failureThreshold is hit
   ```

**Success criteria:** You directly observed that a readiness failure changes `READY` (0/1) without a restart, while a liveness failure increments `RESTARTS` — and can explain why those are different Kubernetes mechanisms, not the same thing.

---

### Lab 4 — LimitRange and ResourceQuota interaction

1. Apply a ResourceQuota requiring explicit CPU/memory requests in the namespace:
   ```bash
   cat <<'EOF' | kubectl apply -f -
   apiVersion: v1
   kind: ResourceQuota
   metadata:
     name: qos-lab-quota
     namespace: qos-lab
   spec:
     hard:
       requests.cpu: "2"
       requests.memory: 2Gi
       pods: "10"
   EOF
   ```
2. Try creating a pod with no resources specified and observe the rejection:
   ```bash
   kubectl run no-resources --image=nginx -n qos-lab
   kubectl get events -n qos-lab --field-selector reason=FailedCreate
   ```
3. Now add a LimitRange supplying defaults, and retry:
   ```bash
   cat <<'EOF' | kubectl apply -f -
   apiVersion: v1
   kind: LimitRange
   metadata:
     name: qos-lab-limits
     namespace: qos-lab
   spec:
     limits:
       - type: Container
         defaultRequest: { cpu: "100m", memory: "128Mi" }
         default: { cpu: "200m", memory: "256Mi" }
   EOF
   kubectl run with-defaults --image=nginx -n qos-lab
   kubectl get pod with-defaults -n qos-lab -o jsonpath='{.spec.containers[0].resources}'
   ```

**Success criteria:** You can explain, from direct observation, why the first pod was rejected and the second succeeded with auto-filled resource values.

---

### Cleanup

```bash
kubectl delete namespace qos-lab
```

### Stretch challenge

Write a `kubectl` one-liner (using `-o jsonpath` or `-o json` + `jq`) that lists every pod in your cluster along with its QoS class, sorted so all `BestEffort` pods appear first — the ones most likely to be evicted first under memory pressure, and therefore the first ones worth reviewing.
