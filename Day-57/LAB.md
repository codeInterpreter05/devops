# Day 57 — Lab: Service Mesh Intro

**Goal:** Install Istio on a local cluster, deploy the Bookinfo sample application, and configure a real 90/10 canary traffic split — the assigned hands-on activity — then compare against Linkerd's simpler install path.

**Prerequisites:** `minikube` (or `kind`) with at least 4 CPUs / 8GB RAM allocated, `kubectl`, `istioctl` installed (`curl -L https://istio.io/downloadIstio | sh -`, then add `istio-<version>/bin` to your PATH), and `linkerd` CLI (`curl -sL https://run.linkerd.io/install | sh`) for the comparison lab.

---

### Lab 1 — Install Istio and verify the control plane

1. Start minikube with enough resources: `minikube start --cpus=4 --memory=8192`
2. Install Istio with the demo profile (enables most features for learning purposes): `istioctl install --set profile=demo -y`
3. Verify: `kubectl get pods -n istio-system` — confirm `istiod` and ingress gateway pods are `Running`.
4. Enable automatic sidecar injection for the default namespace: `kubectl label namespace default istio-injection=enabled`
5. Run `istioctl analyze` — this checks your cluster/mesh config for common misconfigurations before you even deploy anything.

**Success criteria:** `istiod` is Running, and `kubectl get namespace default --show-labels` shows `istio-injection=enabled`.

---

### Lab 2 — Deploy Bookinfo and confirm sidecar injection

1. Deploy the sample app (from the Istio release directory): `kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml`
2. Confirm every pod now has **2/2** containers ready (app + Envoy sidecar): `kubectl get pods`
3. Expose it via the Istio ingress gateway: apply `samples/bookinfo/networking/bookinfo-gateway.yaml`, then get the gateway URL:
   ```bash
   minikube tunnel &     # or use `minikube service istio-ingressgateway -n istio-system --url`
   kubectl get svc istio-ingressgateway -n istio-system
   ```
4. Hit the `/productpage` endpoint repeatedly (`curl` in a loop, or refresh in a browser) and note that the "reviews" section's star ratings differ between requests — Bookinfo ships 3 versions of the `reviews` service (v1: no stars, v2: black stars, v3: red stars) and, with no `VirtualService` yet, the base `Service` load-balances across all three randomly.

**Success criteria:** Every Bookinfo pod shows 2/2 containers, and you can reach `/productpage` through the Istio gateway.

---

### Lab 3 — The core hands-on activity: 90/10 canary traffic split

1. Apply the default destination rules (defines `v1`/`v2`/`v3` subsets for each service): `kubectl apply -f samples/bookinfo/networking/destination-rule-all.yaml`
2. First pin all traffic to `v1` only, to get a stable baseline: apply `samples/bookinfo/networking/virtual-service-all-v1.yaml`. Confirm every refresh of `/productpage` shows the no-stars `reviews` version.
3. Now write your own `VirtualService` for `reviews` splitting 90% to `v1` and 10% to `v3` (red stars — the most visually obvious version to verify against):
   ```yaml
   apiVersion: networking.istio.io/v1beta1
   kind: VirtualService
   metadata: {name: reviews}
   spec:
     hosts: [reviews]
     http:
       - route:
           - destination: {host: reviews, subset: v1}
             weight: 90
           - destination: {host: reviews, subset: v3}
             weight: 10
   ```
4. `kubectl apply -f your-canary-vs.yaml`, then refresh `/productpage` roughly 30-40 times and tally how many show red stars vs. no stars — confirm the ratio is roughly 90/10 (not exact on a small sample, but directionally correct).
5. Use `istioctl analyze` again and `istioctl proxy-config routes <pod-name>.default` to inspect the actual Envoy route config generated from your `VirtualService` — confirm the weights appear in the real Envoy config, proving the abstraction is real, not just a Kubernetes object sitting inert.

**Success criteria:** You can show a live 90/10 (or close to it) split across repeated requests, and you can point to the actual Envoy route config reflecting those weights via `istioctl proxy-config`.

---

### Lab 4 — Circuit breaking and fault injection

1. Add `outlierDetection` to the `reviews` `DestinationRule` (consecutive 5xx ejection) and explain, in writing, what would need to happen for a subset to actually get ejected.
2. Apply a fault-injection rule on the `ratings` service that aborts 20% of requests with a 500, and observe the effect on the Bookinfo UI (the reviews section should show an error for ~1 in 5 refreshes when routed to a version that calls ratings).
3. Remove the fault injection rule afterward — treat leaving it in as you would leave a real production incident.

**Success criteria:** You've triggered and then observed a visible fault-injected failure in the running app, and cleaned it up afterward.

---

### Lab 5 — Compare against Linkerd's install experience

1. On a second, fresh cluster (or after fully uninstalling Istio: `istioctl uninstall --purge -y`), install Linkerd: `linkerd check --pre`, `linkerd install | kubectl apply -f -`, `linkerd check`.
2. Install the viz extension and mesh a simple test deployment (e.g., `kubectl create deployment nginx --image=nginx`, then `kubectl annotate namespace default linkerd.io/inject=enabled` and roll the deployment).
3. Run `linkerd viz stat deploy` and compare, in one paragraph, the install complexity, number of steps, and number of concepts you had to learn versus Istio's install and Bookinfo setup.

**Success criteria:** A written, honest comparison paragraph — this is the artifact that proves you understand the "lighter alternative" tradeoff, not just that you can quote it.

---

### Cleanup

```bash
kubectl delete -f samples/bookinfo/platform/kube/bookinfo.yaml
kubectl delete -f samples/bookinfo/networking/bookinfo-gateway.yaml
kubectl delete -f samples/bookinfo/networking/destination-rule-all.yaml
kubectl delete virtualservice reviews
istioctl uninstall --purge -y
linkerd uninstall | kubectl delete -f -   # if you installed Linkerd
minikube delete
```

### Stretch challenge

Configure header-based routing so that requests with header `end-user: jason` always hit `v2` (black stars) regardless of the 90/10 weighted split for everyone else — then explain, referencing Istio's match-rule evaluation order, why you must place the header-match rule *before* the weighted catch-all rule in the same `VirtualService`.
