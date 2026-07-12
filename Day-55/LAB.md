# Day 55 — Lab: CKA Exam Prep II

**Goal:** Execute the core CKA "practical task" categories end-to-end, under time pressure, using only `etcdctl`, `kubeadm`, and `kubectl` — then run a full timed mock exam.

**Prerequisites:** A `kind` or `kubeadm`-provisioned multi-node cluster (kind is fine for etcd/RBAC/NetworkPolicy tasks; for the upgrade task you need a real kubeadm cluster — a 2-node VM setup, or `kubeadm` inside Vagrant/multipass, since kind clusters don't expose an upgradeable kubeadm control plane the same way). Install `etcdctl` matching your cluster's etcd version, and confirm `kubectl version` matches your cluster.

---

### Lab 1 — etcd backup and restore, for real

1. Locate your etcd static pod manifest and extract the cert paths:
   ```bash
   cat /etc/kubernetes/manifests/etcd.yaml | grep -E 'cert-file|key-file|trusted-ca-file|listen-client-urls'
   ```
2. Take a snapshot:
   ```bash
   ETCDCTL_API=3 etcdctl snapshot save /opt/backup/etcd-snap-$(date +%s).db \
     --endpoints=https://127.0.0.1:2379 \
     --cacert=/etc/kubernetes/pki/etcd/ca.crt \
     --cert=/etc/kubernetes/pki/etcd/server.crt \
     --key=/etc/kubernetes/pki/etcd/server.key
   ```
3. Verify it: `ETCDCTL_API=3 etcdctl --write-out=table snapshot status /opt/backup/etcd-snap-*.db`
4. Break something on purpose: `kubectl create namespace canary-delete-me` then `kubectl delete namespace canary-delete-me` — simulate real data loss by deleting a Deployment you care about, e.g. create `kubectl create deployment nginx-test --image=nginx`, confirm it's running, then delete it: `kubectl delete deployment nginx-test`.
5. Restore from the snapshot taken **before** the deletion into a new data directory, repoint `etcd.yaml`'s hostPath, and confirm the deployment is back:
   ```bash
   ETCDCTL_API=3 etcdctl snapshot restore /opt/backup/etcd-snap-*.db \
     --data-dir=/var/lib/etcd-from-backup
   # edit /etc/kubernetes/manifests/etcd.yaml: change hostPath.path to /var/lib/etcd-from-backup
   # wait for the static pod to restart, then:
   kubectl get deployment nginx-test
   ```

**Success criteria:** `nginx-test` reappears after restore, proving the snapshot captured a consistent prior state and the restore procedure actually rehydrates the cluster.

---

### Lab 2 — Cluster upgrade, one node at a time

1. Check current and available versions: `kubeadm version`, `apt-cache madison kubeadm` (or `kubectl version`).
2. On the first control-plane node: `kubeadm upgrade plan`, then `kubeadm upgrade apply v<next-minor>.0`.
3. Drain, upgrade kubelet/kubectl, uncordon — on the control-plane node itself:
   ```bash
   kubectl drain <cp-node> --ignore-daemonsets --delete-emptydir-data
   apt-mark unhold kubelet kubectl && apt install -y kubelet=<version> kubectl=<version> && apt-mark hold kubelet kubectl
   systemctl daemon-reexec && systemctl restart kubelet
   kubectl uncordon <cp-node>
   ```
4. Repeat drain → upgrade kubelet/kubectl → uncordon on each worker node.
5. Confirm: `kubectl get nodes -o wide` — every node should report the new version under `KUBELET-VERSION`.

**Success criteria:** All nodes show the target minor version, no node was left cordoned, and no downtime-sensitive workload was force-killed (check `kubectl get pods -A -o wide` for anything stuck `Pending`).

---

### Lab 3 — NetworkPolicy: default-deny, then selectively allow

1. Deploy two namespaces with a pod each: `kubectl create ns netpol-lab`, then run a `frontend` and `backend` pod with labels `app=frontend` / `app=backend` (use `kubectl run frontend --image=busybox -n netpol-lab --labels=app=frontend -- sleep 3600` style commands, plus a `backend` pod serving on port 80, e.g. `kubectl run backend --image=nginx -n netpol-lab --labels=app=backend`).
2. Confirm connectivity works with no policy: `kubectl exec -n netpol-lab frontend -- wget -qO- --timeout=2 backend` (use the backend pod's IP if no Service exists).
3. Apply a default-deny-ingress policy scoped to `app=backend`, confirm the same `wget` now times out.
4. Apply an allow-from-frontend policy on top, confirm connectivity is restored.
5. Check which CNI your cluster runs (`kubectl get pods -n kube-system`) and note whether it's one of the enforcing ones (Calico/Cilium/Weave) — if you're on kind's default CNI, confirm whether NetworkPolicy enforcement works or not, and record why.

**Success criteria:** You can demonstrate the before/deny/allow sequence live, and you can explain — from what you observed, not just theory — whether your specific cluster's CNI enforces the policy at all.

---

### Lab 4 — RBAC: least-privilege ServiceAccount

1. Create a namespace `rbac-lab` and a ServiceAccount `ci-bot` inside it.
2. Create a Role granting only `get`, `list`, `watch` on `pods` and `deployments`, and bind it to `ci-bot` via a RoleBinding.
3. Verify with `kubectl auth can-i`:
   ```bash
   kubectl auth can-i list pods --as=system:serviceaccount:rbac-lab:ci-bot -n rbac-lab       # yes
   kubectl auth can-i delete pods --as=system:serviceaccount:rbac-lab:ci-bot -n rbac-lab     # no
   kubectl auth can-i list pods --as=system:serviceaccount:rbac-lab:ci-bot -n default        # no (namespaced!)
   ```
4. Now bind the built-in `view` ClusterRole to `ci-bot` via a namespace-scoped RoleBinding instead of your custom Role, and re-run the same three checks — confirm the outcome and scope match what you'd expect.
5. Generate a short-lived token for `ci-bot` and use it directly with `curl` against the API server to prove the identity is real, not just a `kubectl` abstraction: `kubectl create token ci-bot -n rbac-lab --duration=10m`.

**Success criteria:** You can explain, live, why the third `can-i` check returns "no" even though the first returns "yes" — that's the Role-is-namespaced lesson landing for real.

---

### Lab 5 — The core hands-on activity: full timed CKA mock exam

1. Set a **2-hour timer**. Attempt a full mock exam covering: etcd backup/restore, a cluster upgrade or component fix, a NetworkPolicy task, an RBAC task, and at least one "something is broken, fix it" scenario (a crash-looping static pod from a bad manifest edit is a realistic one to simulate yourself).
2. Score yourself strictly — partial credit only if the object would actually pass a real grader (e.g., wrong namespace = 0, not partial).
3. For every miss, write one sentence: what the *actual* misconception was (not just "I forgot a flag") — e.g., "I didn't realize ClusterRole could be bound namespace-scoped."

**Success criteria:** A written list of misses with root causes, not just a score — the root-cause list is what actually improves your next attempt.

---

### Cleanup

```bash
kubectl delete ns netpol-lab rbac-lab canary-delete-me --ignore-not-found
kubectl delete deployment nginx-test --ignore-not-found
rm -f /opt/backup/etcd-snap-*.db
# if you created a /var/lib/etcd-from-backup directory and want to revert:
# edit /etc/kubernetes/manifests/etcd.yaml back to /var/lib/etcd, then rm -rf /var/lib/etcd-from-backup
```

### Stretch challenge

Simulate total etcd data loss on a **single-node** kubeadm cluster (stop the kubelet, `mv /var/lib/etcd /var/lib/etcd.lost`) and recover purely from your latest snapshot without any other hints — time yourself, and compare against the ~10-15 minutes this task is typically budgeted for on the real exam.
