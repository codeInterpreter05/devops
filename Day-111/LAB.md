# Day 111 — Lab: Cilium & eBPF

**Goal:** Replace kube-proxy with Cilium on a test cluster, enable Hubble, trace real pod-to-pod flows, and write an L7-aware network policy you can prove is working — not just applied.

**Prerequisites:** A disposable test cluster you're comfortable rebuilding (`kind` is ideal since it lets you create a cluster with `kube-proxy` disabled from the start). Install the CLIs:

```bash
# Cilium CLI
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/latest/download/cilium-darwin-arm64.tar.gz
tar xzvf cilium-darwin-arm64.tar.gz && sudo mv cilium /usr/local/bin

# Hubble CLI
curl -L --fail --remote-name-all https://github.com/cilium/hubble/releases/latest/download/hubble-darwin-arm64.tar.gz
tar xzvf hubble-darwin-arm64.tar.gz && sudo mv hubble /usr/local/bin
```

---

### Lab 1 — Create a kube-proxy-free kind cluster

1. Create a kind cluster config that skips the default CNI and kube-proxy:
   ```yaml
   # kind-config.yaml
   kind: Cluster
   apiVersion: kind.x-k8s.io/v1alpha4
   networking:
     disableDefaultCNI: true
     kubeProxyMode: "none"
   ```
2. Create the cluster:
   ```bash
   kind create cluster --config kind-config.yaml --name cilium-lab
   ```
3. Confirm nodes are `NotReady` (expected — no CNI yet):
   ```bash
   kubectl get nodes
   ```

**Success criteria:** Cluster exists, nodes show `NotReady` with a `cni plugin not initialized` condition — proof there's genuinely no CNI running yet, so anything that works afterward is because of Cilium.

---

### Lab 2 — Install Cilium as the CNI, replacing kube-proxy

1. Install with kube-proxy replacement enabled, pointing at the kind API server address:
   ```bash
   API_SERVER_IP=$(docker inspect cilium-lab-control-plane --format '{{ .NetworkSettings.Networks.kind.IPAddress }}')
   cilium install --version 1.15.0 \
     --set kubeProxyReplacement=true \
     --set k8sServiceHost="$API_SERVER_IP" \
     --set k8sServicePort=6443
   ```
2. Wait for and validate the install:
   ```bash
   cilium status --wait
   kubectl get nodes    # should now show Ready
   ```
3. Run Cilium's own connectivity test suite:
   ```bash
   cilium connectivity test
   ```

**Success criteria:** `cilium status` reports all components healthy, nodes are `Ready`, and `cilium connectivity test` passes end to end — this is your evidence that Service routing, pod-to-pod connectivity, and policy enforcement all work purely through eBPF with zero kube-proxy running.

---

### Lab 3 — The core hands-on activity: enable Hubble and trace pod-to-pod flows

This is the assigned hands-on activity for today — do it for real against live traffic, not just a static Hubble UI screenshot.

1. Enable Hubble with the Relay and UI:
   ```bash
   cilium hubble enable --ui
   cilium status --wait
   ```
2. Port-forward and confirm the CLI can reach it:
   ```bash
   cilium hubble port-forward &
   hubble status
   ```
3. Deploy a simple two-pod test (a client curling a server) and watch the flow live:
   ```bash
   kubectl create deployment web --image=nginx
   kubectl expose deployment web --port=80
   kubectl run client --image=busybox --restart=Never -- sleep 3600
   kubectl exec client -- wget -qO- web
   hubble observe --namespace default --follow
   ```
4. In a second terminal, watch flows filtered to just this pair while re-running the `wget`:
   ```bash
   hubble observe --pod default/client --follow
   ```

**Success criteria:** You can see the live flow from `client` to `web` in `hubble observe` output, including the verdict (`FORWARDED`) and identity labels — and can explain what each column in the output means.

---

### Lab 4 — Write and verify an L7-aware CiliumNetworkPolicy

1. Apply a policy that only allows `GET /` from `client` to `web`, denying everything else:
   ```yaml
   apiVersion: cilium.io/v2
   kind: CiliumNetworkPolicy
   metadata:
     name: web-l7-policy
   spec:
     endpointSelector:
       matchLabels:
         app: web
     ingress:
     - fromEndpoints:
       - matchLabels:
           run: client
       toPorts:
       - ports:
         - port: "80"
           protocol: TCP
         rules:
           http:
           - method: "GET"
             path: "/"
   ```
2. Confirm the allowed path still works:
   ```bash
   kubectl exec client -- wget -qO- web
   ```
3. Confirm a disallowed method is dropped and shows up in Hubble:
   ```bash
   kubectl exec client -- wget -qO- --method=POST web 2>&1 || true
   hubble observe --verdict DROPPED --follow
   ```

**Success criteria:** The `GET /` request succeeds, the `POST` request fails, and `hubble observe --verdict DROPPED` shows the exact dropped flow with the policy that caused it — real proof of L7 enforcement, not just "the YAML applied without error."

---

### Cleanup

```bash
kind delete cluster --name cilium-lab
```

### Stretch challenge

Write a Tetragon `TracingPolicy` (observe-only, no `Sigkill`) that logs every `execve` of `/bin/sh` inside the `client` pod's namespace, trigger it by exec-ing into the pod, and find the resulting event with `tetra getevents` — then explain, without running it, what you'd need to change to turn it from an *observability* policy into an *enforcement* policy that kills the process instead.
