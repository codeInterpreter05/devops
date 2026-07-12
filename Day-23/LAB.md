# Day 23 — Lab: Workloads — Deployments & StatefulSets

**Goal:** Deploy and break a real StatefulSet-backed Redis instance, watch rolling updates and rollbacks happen live on a Deployment, and see DaemonSets and lifecycle hooks fire in practice.

**Prerequisites:** A running minikube cluster (`minikube start`) and `kubectl`. `helm` optional for one stretch step.

---

### Lab 1 — Deployment rolling update and rollback

1. Create a Deployment at an old version:
   ```bash
   kubectl create deployment web --image=nginx:1.24 --replicas=4
   kubectl set resources deployment/web --requests=cpu=50m,memory=64Mi
   kubectl patch deployment web -p '{"spec":{"strategy":{"rollingUpdate":{"maxUnavailable":1,"maxSurge":1}}}}'
   kubectl rollout status deployment/web
   ```
2. Watch ReplicaSets before and after an update:
   ```bash
   kubectl get rs -l app=web
   kubectl set image deployment/web nginx=nginx:1.25
   kubectl get rs -l app=web -w      # watch old RS scale to 0 as new RS scales to 4
   ```
3. Check rollout history and roll back:
   ```bash
   kubectl rollout history deployment/web
   kubectl rollout undo deployment/web
   kubectl rollout status deployment/web
   kubectl get rs -l app=web          # confirm the old RS scaled back up instead of a new one being created
   ```
4. Deliberately break a rollout with a bad image tag and observe it stall:
   ```bash
   kubectl set image deployment/web nginx=nginx:this-tag-does-not-exist
   kubectl rollout status deployment/web --timeout=30s   # should time out / report not progressing
   kubectl get pods -l app=web
   kubectl describe pod -l app=web | grep -A5 Events    # ImagePullBackOff / ErrImagePull
   kubectl rollout undo deployment/web
   ```

**Success criteria:** You've watched a new ReplicaSet get created and scaled up while the old one scales down, performed a rollback and confirmed it reused the old ReplicaSet, and seen a broken rollout stall with a clear error rather than silently taking down the service.

---

### Lab 2 — The core hands-on activity: StatefulSet Redis, simulate failure, verify stable DNS

This is the assigned hands-on activity — do it for real.

1. Create a headless Service and a 3-replica Redis StatefulSet:
   ```bash
   cat <<'EOF' | kubectl apply -f -
   apiVersion: v1
   kind: Service
   metadata:
     name: redis
   spec:
     clusterIP: None
     selector:
       app: redis
     ports:
       - port: 6379
   ---
   apiVersion: apps/v1
   kind: StatefulSet
   metadata:
     name: redis
   spec:
     serviceName: redis
     replicas: 3
     selector:
       matchLabels: { app: redis }
     template:
       metadata:
         labels: { app: redis }
       spec:
         containers:
           - name: redis
             image: redis:7
             ports: [{ containerPort: 6379 }]
     volumeClaimTemplates:
       - metadata: { name: data }
         spec:
           accessModes: ["ReadWriteOnce"]
           resources: { requests: { storage: 1Gi } }
   EOF
   kubectl get pods -l app=redis -w      # confirm they come up in order: -0, then -1, then -2
   ```
2. Verify stable DNS names from inside the cluster:
   ```bash
   kubectl run dns-test --image=busybox:1.36 --rm -it --restart=Never -- \
     nslookup redis-1.redis.default.svc.cluster.local
   ```
3. Write a marker value into `redis-1` specifically:
   ```bash
   kubectl exec redis-1 -- redis-cli set marker "hello-from-redis-1"
   kubectl exec redis-1 -- redis-cli get marker
   ```
4. Simulate a pod failure and confirm identity + data survive:
   ```bash
   kubectl delete pod redis-1
   kubectl get pods -l app=redis -w        # wait for redis-1 to come back — same name, same ordinal
   kubectl exec redis-1 -- redis-cli get marker   # should still return "hello-from-redis-1"
   ```
5. Confirm the PVC survived the pod deletion (it's bound to the ordinal, not the pod instance):
   ```bash
   kubectl get pvc -l app=redis
   ```

**Success criteria:** You've observed ordered pod creation (`-0` before `-1` before `-2`), confirmed per-pod DNS resolves, and proven that deleting `redis-1` brings back a pod with the same name *and* the same data, because the PVC and identity are tied to the ordinal, not the pod instance.

---

### Lab 3 — DaemonSet and lifecycle hooks

1. Deploy a simple DaemonSet and confirm one pod per node:
   ```bash
   kubectl create -f - <<'EOF'
   apiVersion: apps/v1
   kind: DaemonSet
   metadata:
     name: node-agent
   spec:
     selector:
       matchLabels: { app: node-agent }
     template:
       metadata:
         labels: { app: node-agent }
       spec:
         tolerations:
           - operator: Exists
         containers:
           - name: agent
             image: busybox:1.36
             command: ["sh", "-c", "sleep 3600"]
   EOF
   kubectl get pods -l app=node-agent -o wide
   kubectl get nodes --no-headers | wc -l    # confirm pod count == node count
   ```
2. Add `postStart`/`preStop` hooks to a throwaway pod and watch them fire:
   ```bash
   cat <<'EOF' | kubectl apply -f -
   apiVersion: v1
   kind: Pod
   metadata:
     name: hook-demo
   spec:
     terminationGracePeriodSeconds: 20
     containers:
       - name: app
         image: busybox:1.36
         command: ["sh", "-c", "echo running; sleep 3600"]
         lifecycle:
           postStart:
             exec: { command: ["sh", "-c", "echo POSTSTART_RAN >> /tmp/lifecycle.log"] }
           preStop:
             exec: { command: ["sh", "-c", "echo PRESTOP_RAN; sleep 10"] }
   EOF
   kubectl exec hook-demo -- cat /tmp/lifecycle.log
   ```
3. Delete the pod and time how long it takes to actually disappear:
   ```bash
   time kubectl delete pod hook-demo
   ```
   It should take roughly 10+ seconds (the `preStop` sleep), not instantly.

**Success criteria:** You've confirmed a DaemonSet pod count exactly matches node count, and observed empirically (via `time`) that `preStop` delays pod termination by the duration of its own command.

---

### Cleanup

```bash
kubectl delete deployment web --ignore-not-found
kubectl delete statefulset redis --ignore-not-found
kubectl delete svc redis --ignore-not-found
kubectl delete pvc -l app=redis --ignore-not-found
kubectl delete daemonset node-agent --ignore-not-found
kubectl delete pod hook-demo --ignore-not-found
```

### Stretch challenge

Scale the `redis` StatefulSet down to 1 replica, then back up to 3 (`kubectl scale statefulset redis --replicas=1` then `--replicas=3`). Check whether `redis-1`'s marker value survived the round trip through 0 replicas of that ordinal existing. Explain why (hint: revisit what actually happens to the PVC on scale-down vs. StatefulSet deletion).
