# Day 64 — Lab: ArgoCD Deep Dive

**Goal:** Set up ArgoCD, deploy an app via GitOps, and practice syncing, self-healing, waves, and rollbacks hands-on — not just read about them.

**Prerequisites:**
- A local Kubernetes cluster (`kind`, `minikube`, or `k3d`) — a small 1-node cluster is enough.
- `kubectl` and `argocd` CLI installed (`brew install argocd`).
- A Git repo (can be a personal GitHub repo) containing simple Kubernetes manifests for a small app — reuse your Phase 1 project app if you have one, or a trivial nginx Deployment + Service is fine for the lab.

---

### Lab 1 — Install ArgoCD and explore the three components

1. Create a cluster and install ArgoCD:
   ```bash
   kind create cluster --name argocd-lab
   kubectl create namespace argocd
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   kubectl -n argocd wait --for=condition=available --timeout=300s deployment --all
   ```
2. List the pods and match each one to the component it represents:
   ```bash
   kubectl -n argocd get pods
   ```
   You should see `argocd-repo-server`, `argocd-application-controller` (a StatefulSet), `argocd-server` (the api-server), plus `argocd-redis` and `argocd-dex-server` (SSO) as supporting pieces.
3. Port-forward and log in:
   ```bash
   kubectl -n argocd port-forward svc/argocd-server 8080:443 &
   PASS=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d)
   argocd login localhost:8080 --username admin --password "$PASS" --insecure
   ```

**Success criteria:** You can name each ArgoCD pod's role (repo-server, application-controller, api-server) and explain in one sentence what each one is/isn't responsible for.

---

### Lab 2 — The core hands-on activity: deploy your app via GitOps

This is today's assigned hands-on activity.

1. Push simple manifests to a Git repo (a `Deployment` + `Service` under `k8s/`).
2. Create the `Application`:
   ```bash
   argocd app create my-app \
     --repo https://github.com/<you>/<repo>.git \
     --path k8s \
     --dest-server https://kubernetes.default.svc \
     --dest-namespace my-app \
     --sync-policy none
   ```
3. Confirm it shows `OutOfSync`/`Missing` before the first sync:
   ```bash
   argocd app get my-app
   ```
4. Sync manually and watch it converge:
   ```bash
   argocd app sync my-app
   argocd app wait my-app
   kubectl -n my-app get pods
   ```

**Success criteria:** Your app is running in the cluster, and `argocd app get my-app` shows both `Synced` and `Healthy`.

---

### Lab 3 — Practice self-heal and drift

1. Enable automated sync with self-heal:
   ```bash
   argocd app set my-app --sync-policy automated --self-heal --auto-prune
   ```
2. Manually drift the cluster:
   ```bash
   kubectl -n my-app scale deployment my-app --replicas=5
   ```
3. Watch ArgoCD detect and revert the drift within seconds:
   ```bash
   kubectl -n my-app get deployment my-app -w
   ```
4. Now delete the manifest for the Service from your Git repo entirely, commit, and push. Confirm ArgoCD (with `auto-prune` on) deletes the Service from the cluster on its next sync.

**Success criteria:** You've personally watched a manual `kubectl scale` get reverted by ArgoCD, and watched a Git deletion prune the corresponding cluster resource.

---

### Lab 4 — Sync waves and a PreSync hook

1. Add a `PreSync` hook Job to your manifests that just echoes a message (standing in for a real migration):
   ```yaml
   apiVersion: batch/v1
   kind: Job
   metadata:
     name: pre-deploy-check
     annotations:
       argocd.argoproj.io/hook: PreSync
       argocd.argoproj.io/hook-delete-policy: HookSucceeded
   spec:
     template:
       spec:
         containers:
           - name: check
             image: busybox
             command: ["sh", "-c", "echo pre-sync check passed"]
         restartPolicy: Never
   ```
2. Commit, push, and trigger a sync. Watch the ArgoCD UI/CLI (`argocd app get my-app`) show the hook Job running and completing **before** the Deployment updates.
3. Confirm the hook Job is auto-deleted after success (`kubectl -n my-app get jobs` should show nothing left).

**Success criteria:** You can see, in the sync operation log, the hook executing in a distinct phase before the main resources.

---

### Lab 5 — Practice rollback

1. Push a deliberately broken change (e.g., an invalid image tag) to Git and sync.
2. Confirm the Application goes `Degraded` (health check fails — pods stuck in `ImagePullBackOff`).
3. Roll back to the previous working revision:
   ```bash
   argocd app history my-app
   argocd app rollback my-app <PREVIOUS_ID>
   ```
4. Confirm the app returns to `Healthy`.

**Success criteria:** You've triggered a real degraded state and recovered it with `argocd app rollback`, and can explain what rollback actually does (re-applies a previous Git revision's rendered manifests — it does not revert your Git history).

---

### Cleanup

```bash
argocd app delete my-app --cascade
kind delete cluster --name argocd-lab
kill %1   # stop the port-forward background job
```

### Stretch challenge

Convert your single `Application` into an `ApplicationSet` using a `list` generator with two "environments" (`staging`, `production`) pointing at two different namespaces on the same cluster, and confirm ArgoCD auto-creates two `Application` resources from the one `ApplicationSet` definition.
