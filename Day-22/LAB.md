# Day 22 — Lab: K8s Architecture Deep Dive

**Goal:** Stand up a real (single-node) cluster, physically see every control-plane component as a running pod, and trace a Pod's journey from `kubectl apply` to `Running` using events and logs — not just diagrams.

**Prerequisites:** `minikube`, `kubectl`, and `k9s` installed.

```bash
# macOS example — adjust for your OS
brew install minikube kubectl derailed/k9s/k9s
minikube version && kubectl version --client && k9s version
```

---

### Lab 1 — Start minikube and inspect the control plane

1. Start a cluster with a driver you have available (Docker driver is easiest cross-platform):
   ```bash
   minikube start --driver=docker --nodes=1
   ```
2. List every pod in `kube-system` — this is the control plane, physically:
   ```bash
   kubectl get pods -n kube-system -o wide
   ```
   You should see `kube-apiserver-minikube`, `etcd-minikube`, `kube-scheduler-minikube`, `kube-controller-manager-minikube`, `kube-proxy-*`, `coredns-*`, and a CNI component (e.g., `storage-provisioner`, `kindnet` or similar depending on driver).
3. Confirm these are **static pods** by checking the manifest files directly:
   ```bash
   minikube ssh
   sudo ls /etc/kubernetes/manifests/
   sudo cat /etc/kubernetes/manifests/etcd.yaml | head -30
   exit
   ```
4. Try to delete the `kube-apiserver` pod via `kubectl` and observe what happens:
   ```bash
   kubectl delete pod kube-apiserver-minikube -n kube-system
   kubectl get pods -n kube-system -w   # watch it come back on its own within seconds
   ```

**Success criteria:** You can point to each control-plane component running as an actual pod, you've confirmed they're static pods managed by the kubelet directly (not via a Deployment/ReplicaSet), and you've observed that deleting one causes the kubelet to immediately recreate it from the manifest file.

---

### Lab 2 — Explore the cluster visually with k9s

1. Launch k9s:
   ```bash
   k9s
   ```
2. Navigate to `kube-system` namespace (type `:pods<Enter>`, then `:ns<Enter>` to switch namespace), select `etcd-minikube`, and press `l` to view logs.
3. Select `kube-scheduler-minikube` and view its logs — look for lines mentioning `Successfully bound pod to node` once you trigger Lab 3 in another terminal.
4. Press `:node<Enter>` to see node conditions (`Ready`, `MemoryPressure`, `DiskPressure`, `PIDPressure`) that the kubelet reports.

**Success criteria:** You can navigate namespaces, view logs, and read node conditions in k9s without needing to fall back to `kubectl` for basic exploration.

---

### Lab 3 — The core hands-on activity: trace a Pod creation through the API

This is the assigned hands-on activity — do it for real.

1. Create a simple pod manifest:
   ```bash
   cat <<'EOF' > /tmp/pod.yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: trace-demo
     labels:
       app: trace-demo
   spec:
     containers:
       - name: nginx
         image: nginx:1.25
         ports:
           - containerPort: 80
   EOF
   ```
2. In one terminal, start watching events **before** you apply:
   ```bash
   kubectl get events --watch --field-selector involvedObject.name=trace-demo
   ```
3. In a second terminal, apply the manifest and immediately check phase:
   ```bash
   kubectl apply -f /tmp/pod.yaml
   kubectl get pod trace-demo -w
   ```
4. Back in the events terminal, you should see, in order: `Scheduled` → `Pulling` → `Pulled` → `Created` → `Started`. Write down which component logged each event (hint: check the `SOURCE`/`REPORTING CONTROLLER` column with `kubectl get events -o wide`).
5. Confirm the scheduler's decision explicitly:
   ```bash
   kubectl get pod trace-demo -o jsonpath='{.spec.nodeName}'
   kubectl describe pod trace-demo | grep -A5 Events
   ```
6. Now simulate a scheduling failure on purpose — request more CPU than the node has:
   ```bash
   kubectl run huge-pod --image=nginx --requests='cpu=1000' --dry-run=client -o yaml > /tmp/huge-pod.yaml
   kubectl apply -f /tmp/huge-pod.yaml
   kubectl describe pod huge-pod | grep -A10 Events
   ```
   You should see a `FailedScheduling` event with reason `Insufficient cpu` — this is the scheduler's Filter phase rejecting every node.

**Success criteria:** You can narrate, using your own terminal output as evidence, exactly which component (apiserver, scheduler, kubelet) produced each event in the pod creation timeline, and you've seen a real scheduling failure and its exact error message.

---

### Lab 4 — etcd, from the inside

1. SSH into the minikube VM/container and inspect etcd's data directly (read-only, informational):
   ```bash
   minikube ssh
   sudo crictl ps | grep etcd
   ```
2. From your host, check what `resourceVersion` your pod is on, then edit it and watch it change:
   ```bash
   kubectl get pod trace-demo -o jsonpath='{.metadata.resourceVersion}'
   kubectl label pod trace-demo test=1
   kubectl get pod trace-demo -o jsonpath='{.metadata.resourceVersion}'
   ```
3. Explain in your own words why the second `resourceVersion` is higher.

**Success criteria:** You've directly observed `resourceVersion` incrementing on a write, connecting the abstract "etcd is versioned" concept to something you triggered yourself.

---

### Cleanup

```bash
kubectl delete pod trace-demo huge-pod --ignore-not-found
rm -f /tmp/pod.yaml /tmp/huge-pod.yaml
minikube stop
# Full teardown if you want to reclaim disk/resources entirely:
# minikube delete
```

### Stretch challenge

Kill the `etcd-minikube` static pod (`kubectl delete pod etcd-minikube -n kube-system`) and, while it's restarting, try running `kubectl get pods` repeatedly. Note what error you get and how long the disruption lasts. Then explain: why did already-running pods (like `trace-demo`, if still present) keep serving traffic during this window, even though the control plane was degraded?
