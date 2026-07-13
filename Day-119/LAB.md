# Day 119 — Lab: Zero Trust & mTLS

**Goal:** Move from "we have NetworkPolicy" to a working, cryptographically-verified zero trust stack — default-deny micro-segmentation, SPIFFE/SPIRE workload identity issuing real SVIDs, cert-manager-issued mTLS between plain services, and Istio's mesh-native mTLS + identity-based authorization.

**Prerequisites:** `kind` and `kubectl`, Docker running, Helm 3. At least 4 CPUs / 8GB RAM free for `kind` — Istio plus SPIRE plus cert-manager on one small cluster is a lot; a single-node `kind` cluster is fine for these labs (SPIRE's Kubernetes quickstart is written against exactly this kind of setup).

```bash
kind create cluster --name zerotrust
kubectl cluster-info --context kind-zerotrust
```

---

### Lab 1 — Zero trust in miniature: default-deny micro-segmentation

1. Install Calico (kind's default CNI does not enforce NetworkPolicy — verify this claim yourself first):
   ```bash
   kind create cluster --name zerotrust-calico --config - <<'EOF'
   kind: Cluster
   apiVersion: kind.x-k8s.io/v1alpha4
   networking:
     disableDefaultCNI: true
   EOF
   kubectl --context kind-zerotrust-calico apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
   ```
2. Create a namespace with two pods and confirm the flat-trust default: `kubectl create ns microseg`, then run a `client` (busybox) and a `server` (nginx) pod, and confirm `client` can `wget` `server`'s pod IP with **no NetworkPolicy applied at all**.
3. Apply the default-deny-all policy from `01-README-Zero-Trust-Principles-And-BeyondCorp.md` scoped to `microseg`. Re-run the `wget` — confirm it now times out.
4. Apply a scoped allow policy permitting only `client` → `server` on port 80. Confirm connectivity is restored, but a third pod with no matching label still cannot reach `server`.

**Success criteria:** You can demonstrate, live, the before/deny/allow sequence, and explain in one sentence why step 2's result is the actual Kubernetes default (not a misconfiguration you introduced).

---

### Lab 2 — The core hands-on activity: SPIRE server + agent, register a workload, fetch a real SVID

1. Clone the official tutorials repo (this tracks the current quickstart manifests better than hand-typing them, since exact CRD/field names shift between SPIRE releases):
   ```bash
   git clone https://github.com/spiffe/spire-tutorials.git
   cd spire-tutorials/k8s/quickstart
   ```
2. Deploy the SPIRE server (namespace, ServiceAccount, ClusterRole, StatefulSet, Service):
   ```bash
   kubectl apply -f spire-namespace.yaml
   kubectl apply -f server-account.yaml
   kubectl apply -f spire-bundle-configmap.yaml
   kubectl apply -f server-cluster-role.yaml
   kubectl apply -f server-configmap.yaml
   kubectl apply -f server-statefulset.yaml
   kubectl apply -f server-service.yaml
   kubectl rollout status statefulset/spire-server -n spire
   ```
3. Deploy the SPIRE agent DaemonSet:
   ```bash
   kubectl apply -f agent-account.yaml
   kubectl apply -f agent-cluster-role.yaml
   kubectl apply -f agent-configmap.yaml
   kubectl apply -f agent-daemonset.yaml
   kubectl rollout status daemonset/spire-agent -n spire
   ```
4. Confirm node attestation succeeded — list registered agents:
   ```bash
   kubectl exec -n spire spire-server-0 -- \
     /opt/spire/bin/spire-server agent list
   ```
5. Register a workload identity for a specific namespace + ServiceAccount (create the `default` ServiceAccount's identity in the `default` namespace, or a custom one — adjust the agent parent ID to the value returned by step 4):
   ```bash
   kubectl exec -n spire spire-server-0 -- \
     /opt/spire/bin/spire-server entry create \
     -spiffeID spiffe://example.org/ns/default/sa/default \
     -parentID spiffe://example.org/spire/agent/k8s_psat/demo-cluster/<agent-id-from-step-4> \
     -selector k8s:ns:default \
     -selector k8s:sa:default
   ```
6. Deploy a workload pod that mounts the SPIRE agent's socket (the tutorial repo's `client-deployment.yaml` does this for you), then fetch its X.509-SVID from inside the pod:
   ```bash
   kubectl exec -n default <client-pod> -c client -- \
     /opt/spire/bin/spire-agent api fetch x509 \
     -socketPath /run/spire/sockets/agent.sock
   ```

**Success criteria:** Step 6 returns a real SPIFFE ID and certificate details for your workload — not an error. Explain, from what you just did (not from the README), which two attestation steps had to both succeed for that SVID to be issued.

---

### Lab 3 — mTLS between two plain services using cert-manager (no mesh)

1. Install cert-manager:
   ```bash
   helm repo add jetstack https://charts.jetstack.io && helm repo update
   helm install cert-manager jetstack/cert-manager \
     --namespace cert-manager --create-namespace --set installCRDs=true
   ```
2. Apply the self-signed root → intermediate CA `ClusterIssuer`/`Certificate`/`ClusterIssuer` chain from `03-README-mTLS-Everywhere-Cert-Manager-Istio.md`.
3. Request two leaf certs with `usages: ["client auth", "server auth"]` — one for a `server` app, one for a `client` app — both issued by `internal-ca-issuer`, each in its own namespace's Secret.
4. Configure a simple `nginx` server to require client certs (`ssl_verify_client on;` and `ssl_client_certificate` pointing at the CA bundle from the `internal-ca-key-pair` Secret) and a `curl --cert --key --cacert` call from the client pod. Confirm the connection succeeds with the client cert, and confirm it's **rejected** when you retry `curl` with `--cert`/`--key` omitted.
5. `kubectl delete secret <server-cert-secret>` and confirm cert-manager recreates it automatically within `renewBefore` behavior (or immediately, since deletion triggers re-issuance) — this demonstrates the "no manual rotation" property.

**Success criteria:** You can show mTLS actually being enforced (unauthenticated `curl` rejected, authenticated `curl` accepted) between two workloads with zero relationship to any service mesh.

---

### Lab 4 — Istio STRICT mTLS + identity-based AuthorizationPolicy

1. Install Istio with the demo profile:
   ```bash
   istioctl install --set profile=demo -y
   kubectl label namespace default istio-injection=enabled
   ```
2. Deploy `frontend` and `payment-service` sample apps (Istio's `samples/httpbin` and `samples/sleep` work fine as stand-ins) into `default`, confirming sidecars were injected (`kubectl get pod -o jsonpath='{.spec.containers[*].name}'` shows `istio-proxy`).
3. Apply mesh-wide `PERMISSIVE` first, confirm both plaintext and mTLS work, then flip to namespace-scoped `STRICT` from the README and confirm a plaintext `curl` from a non-mesh pod (no sidecar) now fails while sidecar-to-sidecar traffic keeps working.
4. Apply the `AuthorizationPolicy` restricting `payment-service` to only accept calls from `frontend`'s principal. Confirm `sleep`/a third unrelated pod gets a 403 even over mTLS, while `frontend` succeeds.
5. `kubectl exec` into the `istio-proxy` sidecar of `payment-service` and inspect the actual peer certificate Envoy received (`openssl s_client` against the sidecar's port, or `istioctl proxy-config secret <pod>`) to see the SPIFFE URI in the SAN field with your own eyes.

**Success criteria:** You can point at the exact SPIFFE ID string inside a live certificate and explain how it maps to the `principals` field in your `AuthorizationPolicy`.

---

### Cleanup

```bash
kind delete cluster --name zerotrust
kind delete cluster --name zerotrust-calico
```

### Stretch challenge

Stand up a **second** SPIRE server representing a different trust domain (`spiffe://partner.example.org`), export both servers' trust bundles, and configure SPIFFE Federation between them so a workload registered in `example.org` can validate an SVID presented by a workload in `partner.example.org` during a manual mTLS handshake (`openssl` is fine for the handshake itself). Explain, in writing, why this is architecturally the same pattern as AWS IRSA/GCP Workload Identity Federation even though neither of those is SPIFFE-branded.
