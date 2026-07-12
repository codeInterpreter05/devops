# Day 41 — Lab: NetworkPolicies

**Goal:** Implement zero-trust NetworkPolicies for a 3-tier app (web → api → db) with a monitoring exemption, on a cluster with a CNI plugin that actually enforces NetworkPolicy — and prove enforcement is real by watching traffic get blocked and unblocked live.

**Prerequisites:**
- A Kubernetes cluster with a NetworkPolicy-enforcing CNI. `kind` clusters do **not** enforce NetworkPolicy by default — you must explicitly install Calico or Cilium, or use `minikube start --cni=calico`. Instructions below use `kind` + Calico.
- `kubectl` and `helm` installed.

---

### Lab 1 — Stand up a cluster with real NetworkPolicy enforcement

1. Create a `kind` cluster with the default CNI disabled so you can install Calico yourself:
   ```bash
   cat > kind-config.yaml <<'EOF'
   kind: Cluster
   apiVersion: kind.x-k8s.io/v1alpha4
   networking:
     disableDefaultCNI: true
   EOF
   kind create cluster --config kind-config.yaml
   ```
2. Install Calico:
   ```bash
   kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml
   kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml
   kubectl get pods -n calico-system -w   # wait until all Running
   ```
3. Confirm nodes are Ready:
   ```bash
   kubectl get nodes
   ```

**Success criteria:** All Calico pods are Running, and `kubectl get nodes` shows every node `Ready` (not `NotReady`, which would indicate the CNI isn't actually up).

---

### Lab 2 — Deploy the 3-tier app and confirm it's fully open by default

1. Create namespace and deployments:
   ```bash
   kubectl create namespace shop
   kubectl -n shop create deployment web --image=nginx:1.25
   kubectl -n shop create deployment api --image=hashicorp/http-echo -- -text="api-ok" -listen=:8080
   kubectl -n shop create deployment db --image=postgres:16 --port=5432
   kubectl -n shop set env deployment/db POSTGRES_PASSWORD=labpass
   kubectl -n shop label deployment web app=web --overwrite
   kubectl -n shop label deployment api app=api --overwrite
   kubectl -n shop label deployment db app=db --overwrite
   kubectl -n shop expose deployment api --port=8080
   kubectl -n shop expose deployment db --port=5432
   ```
2. Confirm `web` can currently reach `db` directly (proving the default-open problem):
   ```bash
   kubectl -n shop run test-web --rm -it --image=busybox --labels="app=web" -- sh -c "nc -zv db 5432"
   ```

**Success criteria:** The connection to `db:5432` succeeds from a `web`-labeled pod, proving the network is wide open before any policy exists.

---

### Lab 3 — The core hands-on activity: implement zero-trust with a monitoring exemption

1. Apply default-deny-all for the namespace:
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata: { name: default-deny-all, namespace: shop }
   spec:
     podSelector: {}
     policyTypes: [Ingress, Egress]
   ```
2. Allow DNS egress for everything (or every pod loses name resolution):
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata: { name: allow-dns, namespace: shop }
   spec:
     podSelector: {}
     policyTypes: [Egress]
     egress:
       - to: [{ namespaceSelector: {} }]
         ports: [{ protocol: UDP, port: 53 }, { protocol: TCP, port: 53 }]
   ```
3. Allow `web -> api` on 8080, and `api -> db` on 5432 (ingress-only, per file 2's pattern), plus matching egress for `web` and `api`:
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata: { name: allow-web-to-api, namespace: shop }
   spec:
     podSelector: { matchLabels: { app: api } }
     policyTypes: [Ingress]
     ingress:
       - from: [{ podSelector: { matchLabels: { app: web } } }]
         ports: [{ protocol: TCP, port: 8080 }]
   ---
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata: { name: allow-api-to-db, namespace: shop }
   spec:
     podSelector: { matchLabels: { app: db } }
     policyTypes: [Ingress]
     ingress:
       - from: [{ podSelector: { matchLabels: { app: api } } }]
         ports: [{ protocol: TCP, port: 5432 }]
   ---
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata: { name: allow-web-egress, namespace: shop }
   spec:
     podSelector: { matchLabels: { app: web } }
     policyTypes: [Egress]
     egress:
       - to: [{ podSelector: { matchLabels: { app: api } } }]
         ports: [{ protocol: TCP, port: 8080 }]
   ---
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata: { name: allow-api-egress, namespace: shop }
   spec:
     podSelector: { matchLabels: { app: api } }
     policyTypes: [Egress]
     egress:
       - to: [{ podSelector: { matchLabels: { app: db } } }]
         ports: [{ protocol: TCP, port: 5432 }]
   ```
4. Re-run the Lab 2 test and confirm `web -> db` direct connection now **fails**:
   ```bash
   kubectl -n shop run test-web --rm -it --image=busybox --labels="app=web" -- sh -c "nc -zv -w3 db 5432"
   ```
5. Confirm `web -> api` still works:
   ```bash
   kubectl -n shop run test-web --rm -it --image=busybox --labels="app=web" -- sh -c "nc -zv -w3 api 8080"
   ```
6. Add the monitoring exemption:
   ```bash
   kubectl create namespace monitoring
   kubectl -n monitoring label namespace monitoring kubernetes.io/metadata.name=monitoring --overwrite
   ```
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata: { name: allow-monitoring-scrape, namespace: shop }
   spec:
     podSelector: {}
     policyTypes: [Ingress]
     ingress:
       - from:
           - namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: monitoring } }
             podSelector: { matchLabels: { app: prometheus } }
         ports: [{ protocol: TCP, port: 8080 }]
   ```
7. Simulate a Prometheus pod and confirm it can now reach `api`, while a random unlabeled pod in the `default` namespace still cannot:
   ```bash
   kubectl -n monitoring run fake-prometheus --rm -it --image=busybox --labels="app=prometheus" -- sh -c "nc -zv -w3 api.shop.svc.cluster.local 8080"
   kubectl run test-random --rm -it --image=busybox -- sh -c "nc -zv -w3 api.shop.svc.cluster.local 8080"
   ```

**Success criteria:** `web -> db` direct access is blocked, `web -> api` and `api -> db` work, the labeled fake-Prometheus pod from `monitoring` can reach `api`, and an unrelated unlabeled pod cannot.

---

### Cleanup

```bash
kubectl delete namespace shop monitoring
kind delete cluster
```

### Stretch challenge

Add a `CiliumNetworkPolicy`-equivalent restriction (if you swap Calico for Cilium in Lab 1) that allows `web -> api` only for `GET` requests on `/api/v1/*`, and prove a `POST` request is blocked while a `GET` succeeds — demonstrating Layer 7 policy enforcement that standard NetworkPolicy cannot express.
