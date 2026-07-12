# Day 40 — Lab: K8s Observability & Debugging

**Goal:** Intentionally break a deployment 5 different ways and practice diagnosing and fixing each from scratch, using only the debugging toolkit from today's notes — no peeking at the "answer" until you've genuinely tried `describe`/`logs`/exit codes first.

**Prerequisites:**
- A local Kubernetes cluster (`kind`/`minikube`) with `kubectl`.
- `k9s` installed (`brew install k9s`) and `stern` installed (`brew install stern`).
- A baseline working Deployment to break — use this as your starting point:
  ```bash
  kubectl create deployment myapp --image=nginx:1.25 --replicas=1
  kubectl expose deployment myapp --port=80
  kubectl rollout status deployment/myapp
  ```

---

### Lab 1 — Break it with a bad image tag (`ImagePullBackOff`)

1. Break it:
   ```bash
   kubectl set image deployment/myapp myapp=nginx:this-tag-does-not-exist
   ```
2. Diagnose using only `kubectl describe pod` and the Events section — do not guess, read the exact message.
3. Fix it:
   ```bash
   kubectl set image deployment/myapp myapp=nginx:1.25
   kubectl rollout status deployment/myapp
   ```

**Success criteria:** You can state the exact Events message that revealed the cause, verbatim, before applying the fix.

---

### Lab 2 — Break it with an OOM kill

1. Constrain memory far below what nginx needs and generate load:
   ```bash
   kubectl set resources deployment/myapp -c=myapp --limits=memory=8Mi --requests=memory=8Mi
   ```
2. Watch it get OOMKilled:
   ```bash
   kubectl get pods -w
   kubectl describe pod -l app=myapp | grep -A5 "Last State"
   ```
3. Confirm the exit code is 137 and the Reason is `OOMKilled`.
4. Fix it:
   ```bash
   kubectl set resources deployment/myapp -c=myapp --limits=memory=128Mi --requests=memory=64Mi
   ```

**Success criteria:** You've personally seen exit code 137 and `Reason: OOMKilled` in `describe` output, not just read about it.

---

### Lab 3 — Break it with a bad ConfigMap/Secret reference (`CreateContainerConfigError`)

1. Patch the deployment to reference a Secret key that doesn't exist:
   ```bash
   kubectl patch deployment myapp --type=json -p='[{"op":"add","path":"/spec/template/spec/containers/0/env","value":[{"name":"MYVAR","valueFrom":{"secretKeyRef":{"name":"nonexistent-secret","key":"password"}}}]}]'
   ```
2. Diagnose with `describe` — identify the exact missing reference from the Events text.
3. Fix it by creating the missing secret, or reverting the patch:
   ```bash
   kubectl patch deployment myapp --type=json -p='[{"op":"remove","path":"/spec/template/spec/containers/0/env"}]'
   ```

**Success criteria:** You can explain, from the Events output alone, exactly which Secret and which key was missing.

---

### Lab 4 — Break it with a failing init container

1. Add an init container that will never succeed:
   ```bash
   kubectl patch deployment myapp --type=json -p='[{"op":"add","path":"/spec/template/spec/initContainers","value":[{"name":"wait-for-nothing","image":"busybox","command":["sh","-c","exit 1"]}]}]'
   ```
2. Observe the pod stuck in `Init:Error` or `Init:CrashLoopBackOff`.
3. Confirm `kubectl logs <pod>` (no `-c`) shows nothing useful, then get the real info:
   ```bash
   kubectl logs <pod> -c wait-for-nothing
   kubectl describe pod <pod>
   ```
4. Fix it by removing the bad init container:
   ```bash
   kubectl patch deployment myapp --type=json -p='[{"op":"remove","path":"/spec/template/spec/initContainers"}]'
   ```

**Success criteria:** You can explain why the main container's logs were empty/irrelevant while the pod was stuck on this init container.

---

### Lab 5 — Break it with an overly aggressive liveness probe

1. Add a liveness probe that will fail immediately on a slow-enough-seeming startup:
   ```bash
   kubectl patch deployment myapp --type=json -p='[{"op":"add","path":"/spec/template/spec/containers/0/livenessProbe","value":{"httpGet":{"path":"/does-not-exist","port":80},"initialDelaySeconds":0,"periodSeconds":2,"failureThreshold":1}}]'
   ```
2. Watch the pod enter `CrashLoopBackOff` purely from Kubernetes killing it, not the app crashing on its own — confirm via `describe` Events showing "Liveness probe failed."
3. Fix it:
   ```bash
   kubectl patch deployment myapp --type=json -p='[{"op":"remove","path":"/spec/template/spec/containers/0/livenessProbe"}]'
   ```

**Success criteria:** You can articulate, from direct observation, why this looks identical to an application crash from `kubectl get pods` alone, and what specifically in `describe`'s Events distinguishes a probe-induced kill from a real app crash.

---

### Bonus: use `k9s` and `stern` for the same investigations

Repeat Lab 2 or Lab 5 using `k9s` (navigate to the pod, press `d` for describe, `l` for logs, `s` for shell) and `stern myapp` to tail logs across restarts — compare the experience to raw `kubectl` commands.

---

### Cleanup

```bash
kubectl delete deployment myapp
kubectl delete service myapp
```

### Stretch challenge

Write a single bash script that takes a pod name and automatically runs `describe`, `logs --previous`, and prints a best-guess diagnosis (checking for exit code 137, "ImagePull" in events, "Init:" in status, etc.) — a miniature version of the mental triage table from file 3.
