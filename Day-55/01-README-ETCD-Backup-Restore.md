# Day 55 — CKA Exam Prep II: ETCD Backup & Restore

**Phase:** 1 – Core DevOps | **Week:** W9 | **Domain:** Certifications | **Flag:** 📌

## Brief

If you take the CKA exam, you will almost certainly be asked to back up and restore etcd under time pressure — it is one of the highest-weighted, most-tested tasks on the exam because it maps directly to a real production skill: **etcd is the single source of truth for the entire cluster state**. Every Deployment, Secret, ConfigMap, Service, and RBAC binding you have ever created lives as a key in etcd. Lose etcd without a backup and you have lost the cluster, full stop — the nodes, pods, and running containers are still there, but nothing can be reconciled, scaled, or recovered without re-creating every object by hand. This note builds the muscle memory for the backup/restore flow itself; the next two notes cover cluster upgrades, NetworkPolicy, RBAC, and debugging — the other heavily-tested CKA task categories.

This day is split into three focused files:

1. **This file** — etcd's role, `etcdctl` backup/restore mechanics, snapshot verification.
2. **[02-README-Cluster-Upgrades-NetworkPolicy.md](02-README-Cluster-Upgrades-NetworkPolicy.md)** — `kubeadm upgrade` flow and NetworkPolicy task patterns.
3. **[03-README-RBAC-Troubleshooting.md](03-README-RBAC-Troubleshooting.md)** — RBAC task patterns and cluster-level debugging.

## Why etcd is the thing you back up (not just "the database")

Kubernetes is fundamentally a **declarative reconciliation system** built on top of a key-value store. The API server is stateless — it validates and serializes requests, then reads/writes to etcd. Controllers (kube-controller-manager, kubelet, scheduler) all watch etcd (via the API server's watch mechanism) and drive the *actual* state toward the *desired* state stored there. If etcd is gone:

- The API server has nothing to serve — `kubectl get pods` returns nothing or errors.
- Running containers on nodes **keep running** (kubelet doesn't need the API server to keep existing containers alive), but nothing can be rescheduled, scaled, or healed.
- There is no record of Secrets, RBAC bindings, or any object you created — you'd have to reconstruct the entire cluster manifest by hand, assuming you had it version-controlled somewhere (GitOps repos are your real safety net; etcd backups are the fallback for *cluster-internal* state).

This is why "take a backup of etcd" is effectively "take a backup of the cluster."

## `etcdctl` — the tool, and the flags that trip people up

`etcdctl` is etcd's own CLI client — it talks directly to the etcd gRPC API, not to the Kubernetes API server. That distinction matters: you use `etcdctl` when the API server itself might be unavailable or when you need cluster-internal-state operations that `kubectl` has no concept of.

Two environment details you must get right every time:

```bash
# etcdctl defaults to API v2 unless told otherwise — always force v3
export ETCDCTL_API=3

# etcdctl needs TLS certs to talk to etcd's secured client port (usually 2379)
etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list
```

Find those cert paths fast: they're always defined as flags inside the static pod manifest at `/etc/kubernetes/manifests/etcd.yaml` (look for `--cert-file`, `--key-file`, `--trusted-ca-file`, and `--listen-client-urls` to confirm the port). Under exam pressure, `cat /etc/kubernetes/manifests/etcd.yaml | grep -A1 'cert\|key\|trusted-ca\|listen-client'` gets you every flag you need in one shot instead of guessing paths.

## Taking a snapshot

```bash
ETCDCTL_API=3 etcdctl snapshot save /opt/backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

`snapshot save` takes a point-in-time, consistent copy of the entire etcd keyspace using etcd's Raft log — it does **not** lock or pause the cluster while it runs. Always verify immediately after:

```bash
ETCDCTL_API=3 etcdctl --write-out=table snapshot status /opt/backup/etcd-snapshot.db
```

`snapshot status` reads the file's embedded hash, revision, and key count *without* needing any of the connection flags (it reads a local file, not the live cluster) — a common exam gotcha is passing `--endpoints`/certs to `snapshot status` when they're not needed and not required.

## Restoring a snapshot — the part people get wrong

Restoring does **not** write back into the running etcd cluster. `etcdctl snapshot restore` creates a **brand-new data directory** from the snapshot file — you then have to point etcd's static pod manifest at that new directory:

```bash
ETCDCTL_API=3 etcdctl snapshot restore /opt/backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restored \
  --name=<etcd-member-name> \
  --initial-cluster=<etcd-member-name>=https://127.0.0.1:2380 \
  --initial-advertise-peer-urls=https://127.0.0.1:2380
```

Then, the actual cutover:

1. Edit `/etc/kubernetes/manifests/etcd.yaml` — change the `hostPath` volume that currently points at `/var/lib/etcd` to point at `/var/lib/etcd-restored` instead (both the volume definition and the `--data-dir` flag if present).
2. Because `/etc/kubernetes/manifests/` is watched by the **static pod mechanism** on the kubelet, saving the edited manifest automatically kills and recreates the etcd pod pointing at the new data directory — you don't `kubectl apply` anything, and you don't manually restart a service.
3. Watch it come back: `crictl ps | grep etcd` (or `docker ps` on older setups) and `kubectl get pods -n kube-system` once the API server reconnects.

**Why this two-step dance exists:** etcd's Raft protocol needs a coherent cluster membership state to start correctly; restoring in-place over a live data directory risks corrupting Raft's log/term bookkeeping. Restoring to a fresh directory and swapping the mount is the only sanctioned path — never `rm -rf /var/lib/etcd/*` and restore into it directly.

## Points to Remember

- `ETCDCTL_API=3` first, every time — v2 is still the silent default in many images and its command syntax is different and mostly deprecated.
- Cert/key/CA paths and the client port always come from `/etc/kubernetes/manifests/etcd.yaml` — don't memorize paths, know where to look them up in seconds.
- `snapshot save` is non-disruptive and safe to run repeatedly; `snapshot status` needs no connection flags because it reads a local file.
- Restore always targets a **new** data directory, never overwrites the live one — then you repoint the static pod manifest's volume, you don't restart a service manually.
- Static pod manifests in `/etc/kubernetes/manifests/` are watched by the kubelet — saving a file there is the trigger for pod recreation, which is why editing etcd's manifest "just works" without extra commands.

## Common Mistakes

- Forgetting `ETCDCTL_API=3` and getting confusing "unknown command" errors from v2-style syntax, or worse, v2 silently succeeding against the wrong keyspace.
- Passing all the TLS flags to `snapshot status` (unnecessary — it's a local file read) or forgetting them entirely on `snapshot save`/`restore` against the live endpoint (required — it's a network call).
- Restoring into the same `--data-dir` that etcd is currently using, corrupting the live cluster instead of creating an independent recovery target.
- Editing `etcd.yaml`'s `hostPath` but forgetting to also update the `--data-dir` flag (or vice versa) — the pod comes back reading the old directory because the flag and the mount disagree.
- Not verifying the restored cluster afterward (`kubectl get nodes`, `kubectl get pods -A`, `etcdctl endpoint health`) and assuming "the pod is Running" means "the restore worked."
