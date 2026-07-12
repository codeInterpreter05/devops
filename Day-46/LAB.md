# Day 46 — Lab: K8s Operators & CRDs

**Goal:** Install a real, production-grade Operator (CloudNativePG) and deploy an HA Postgres cluster purely by writing a custom resource — then break it on purpose to see the Operator's failover logic in action.

**Prerequisites:** A Kubernetes cluster (kind/minikube is fine for this lab) with `kubectl` access and cluster-admin-equivalent permissions (installing CRDs and a cluster-scoped controller requires broad RBAC), `helm` installed, and at least 3 schedulable nodes/enough capacity for 3 Postgres pods.

---

### Lab 1 — Install the CloudNativePG Operator

1. Install via the official manifest (simplest) or Helm:
   ```bash
   kubectl apply --server-side -f \
     https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/main/releases/cnpg-1.24.0.yaml
   ```
2. Verify the Operator and its CRDs are installed:
   ```bash
   kubectl get pods -n cnpg-system
   kubectl get crd | grep cnpg.io
   ```
3. Inspect one CRD's schema directly to confirm it's just a schema at this point, nothing running yet:
   ```bash
   kubectl explain cluster.spec --api-version=postgresql.cnpg.io/v1
   ```

**Success criteria:** The `cnpg-controller-manager` pod is `Running` in `cnpg-system`, and `kubectl get crd` lists `clusters.postgresql.cnpg.io` and related CRDs.

---

### Lab 2 — Deploy an HA Postgres cluster using the CRD (the assigned hands-on activity)

1. Apply a 3-instance cluster:
   ```yaml
   apiVersion: postgresql.cnpg.io/v1
   kind: Cluster
   metadata: {name: pg-cluster}
   spec:
     instances: 3
     storage: {size: 1Gi}
     bootstrap:
       initdb: {database: app, owner: app}
   ```
2. Watch the Operator provision it in real time:
   ```bash
   kubectl get cluster pg-cluster -w
   kubectl get pods -l cnpg.io/cluster=pg-cluster -o wide
   ```
3. Identify the current primary:
   ```bash
   kubectl get cluster pg-cluster -o jsonpath='{.status.currentPrimary}'
   ```
4. Connect and confirm the database is real and usable:
   ```bash
   kubectl exec -it pg-cluster-1 -- psql -U app -d app -c "SELECT 1;"
   ```

**Success criteria:** 3 pods running, `status.currentPrimary` populated, and a successful `psql` query against the primary.

---

### Lab 3 — Trigger a failover and observe the reconciliation loop

1. Note the current primary pod name, then delete it directly to simulate a crash:
   ```bash
   kubectl delete pod <current-primary-pod> --grace-period=0 --force
   ```
2. Immediately watch events and the cluster status:
   ```bash
   kubectl get events --sort-by=.lastTimestamp -w
   kubectl get cluster pg-cluster -w
   ```
3. Confirm a different pod was promoted to primary (`status.currentPrimary` changed) and that the deleted pod's replacement rejoined as a standby.
4. Confirm the read-write Service transparently now points at the new primary:
   ```bash
   kubectl get svc pg-cluster-rw -o jsonpath='{.spec.selector}'
   kubectl exec -it pg-cluster-2 -- psql -h pg-cluster-rw -U app -d app -c "SELECT 1;"
   ```

**Success criteria:** You observe an automatic failover completing without manual intervention, and can state the measured time from pod deletion to a new primary being ready.

---

### Lab 4 — Scale the cluster declaratively

1. Edit the `Cluster` object to change `instances: 3` to `instances: 4`:
   ```bash
   kubectl patch cluster pg-cluster --type merge -p '{"spec":{"instances":4}}'
   ```
2. Watch the Operator provision and join the new replica automatically, with no manual `pg_basebackup`/join steps.

**Success criteria:** A 4th pod appears, becomes `Running`, and joins as a streaming replica — confirmed via `kubectl get cluster pg-cluster -o jsonpath='{.status.instances}'`.

---

### Cleanup

```bash
kubectl delete cluster pg-cluster
kubectl delete -f https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/main/releases/cnpg-1.24.0.yaml
```

### Stretch challenge

Configure the `Cluster`'s `backup.barmanObjectStore` section against a local MinIO instance (or a real S3 bucket) running in-cluster, trigger a manual `Backup` custom resource, then delete and restore the cluster from that backup via `bootstrap.recovery` — proving out the full PITR path end-to-end, the same way you'd validate it for RDS in Day 44.
