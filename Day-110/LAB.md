# Day 110 — Lab: Istio Deep Dive

**Goal:** Deploy Istio on a real (or local) cluster, inject the Bookinfo sample app, prove mTLS is actually active, and configure retries/circuit breaking well enough to explain — and demonstrate — how each piece behaves under failure.

**Prerequisites:** A Kubernetes cluster (`kind`, `minikube`, or a real cluster) with at least 4 CPUs / 8GB RAM available, `kubectl` configured against it, and `istioctl` installed:

```bash
curl -L https://istio.io/downloadIstio | sh -
export PATH="$PWD/istio-<version>/bin:$PATH"
istioctl version --remote=false
```

---

### Lab 1 — Install Istio and deploy Bookinfo

1. Install the demo profile (includes ingress gateway, telemetry defaults — good for learning, not production-sized):
   ```bash
   istioctl install --set profile=demo -y
   kubectl label namespace default istio-injection=enabled
   ```
2. Deploy the Bookinfo sample app (ships with the Istio release download):
   ```bash
   kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
   kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
   ```
3. Confirm every pod has 2 containers (app + `istio-proxy`):
   ```bash
   kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].name}{"\n"}{end}'
   ```
4. Get the ingress gateway URL and confirm the app is reachable:
   ```bash
   export INGRESS_HOST=$(kubectl -n istio-system get svc istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
   export INGRESS_PORT=$(kubectl -n istio-system get svc istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
   curl -s "http://$INGRESS_HOST:$INGRESS_PORT/productpage" | grep -o "<title>.*</title>"
   ```

**Success criteria:** `productpage` loads through the ingress gateway, and every Bookinfo pod shows two containers, confirming sidecar injection worked automatically.

---

### Lab 2 — Verify and enforce mTLS between services

1. Before applying any `PeerAuthentication`, confirm the mesh default and check current traffic encryption with:
   ```bash
   istioctl x describe pod "$(kubectl get pod -l app=reviews,version=v1 -o jsonpath='{.items[0].metadata.name}')"
   ```
2. Apply mesh-wide STRICT mTLS:
   ```bash
   kubectl apply -f - <<EOF
   apiVersion: security.istio.io/v1
   kind: PeerAuthentication
   metadata:
     name: default
     namespace: istio-system
   spec:
     mtls:
       mode: STRICT
   EOF
   ```
3. Prove it's active without a UI: run a plaintext `curl` from a non-meshed pod against a meshed service's pod IP directly (should fail/hang) versus from a meshed pod (should succeed transparently).
4. Confirm with `istioctl`:
   ```bash
   istioctl x authz check "$(kubectl get pod -l app=reviews,version=v1 -o jsonpath='{.items[0].metadata.name}')"
   ```

**Success criteria:** You can state, with command output as evidence, that plaintext connections between meshed pods are rejected and mTLS connections succeed — not just "the YAML says STRICT."

---

### Lab 3 — The core hands-on activity: circuit breaking and retry policies

This is the assigned hands-on activity for today — configure it for real and break it on purpose.

1. Apply a `DestinationRule` with a tight connection pool and outlier detection on the `httpbin`-style `reviews` service:
   ```bash
   kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1
   kind: DestinationRule
   metadata:
     name: reviews-cb
   spec:
     host: reviews
     trafficPolicy:
       connectionPool:
         tcp:
           maxConnections: 2
         http:
           http1MaxPendingRequests: 1
           maxRequestsPerConnection: 1
       outlierDetection:
         consecutive5xxErrors: 1
         interval: 10s
         baseEjectionTime: 30s
         maxEjectionPercent: 100
   EOF
   ```
2. Generate concurrent load with `fortio` (or `hey`) to trip the breaker:
   ```bash
   kubectl exec deploy/productpage-v1 -c istio-proxy -- pilot-agent request GET http://reviews:9080/reviews/0 --profile
   # or, if fortio is installed locally against the ingress:
   fortio load -c 10 -qps 0 -n 200 "http://$INGRESS_HOST:$INGRESS_PORT/productpage"
   ```
3. Check breaker activity in Envoy's own stats:
   ```bash
   kubectl exec "$(kubectl get pod -l app=productpage -o jsonpath='{.items[0].metadata.name}')" -c istio-proxy -- \
     pilot-agent request GET stats | grep reviews | grep -E "upstream_rq_pending_overflow|outlier_detection"
   ```
4. Now add retries with a bounded timeout on the `reviews` `VirtualService`:
   ```yaml
   apiVersion: networking.istio.io/v1
   kind: VirtualService
   metadata:
     name: reviews
   spec:
     hosts:
     - reviews
     http:
     - route:
       - destination:
           host: reviews
       retries:
         attempts: 3
         perTryTimeout: 2s
         retryOn: 5xx,connect-failure
       timeout: 5s
   ```
   Apply it, then re-run the load test and observe how the combination of circuit breaking (which sheds excess load) and retries (which re-attempts failed calls) changes the error rate versus latency tradeoff.

**Success criteria:** You can point at `upstream_rq_pending_overflow` (connection-pool rejections) and `outlier_detection.ejections_active` counters going non-zero under load, and explain in your own words why adding retries on top of an already-tripped breaker can make things worse before it makes them better.

---

### Lab 4 — Observability: watch it happen in Kiali and Jaeger

1. Install the addons and open Kiali:
   ```bash
   kubectl apply -f samples/addons
   istioctl dashboard kiali
   ```
2. Generate some traffic (reload `productpage` a dozen times or replay the `fortio` command from Lab 3), then find the live traffic graph in Kiali and confirm the padlock icons on edges between meshed services.
3. Open Jaeger (`istioctl dashboard jaeger`) and find a trace for a single `productpage` request — confirm it shows the full call chain (`productpage` → `reviews` → `ratings`).
4. Deliberately break header propagation: describe (don't implement unless you have time) what would need to change in a hypothetical service that constructs a *new* outbound request without copying incoming headers, and what the resulting trace would look like in Jaeger.

**Success criteria:** You can navigate to a specific trace in Jaeger for a specific request and explain, using Kiali's graph, which edges are encrypted.

---

### Cleanup

```bash
kubectl delete -f samples/addons
kubectl delete -f samples/bookinfo/networking/bookinfo-gateway.yaml
kubectl delete -f samples/bookinfo/platform/kube/bookinfo.yaml
kubectl delete peerauthentication default -n istio-system
kubectl delete destinationrule reviews-cb
istioctl uninstall --purge -y
kubectl label namespace default istio-injection-
```

### Stretch challenge

Configure a `VirtualService` fault-injection rule that returns an HTTP 500 for 50% of requests to the `ratings` service, then observe in Kiali/Jaeger how the circuit breaker and retry policy you configured in Lab 3 react to injected failures rather than real ones — and tune `outlierDetection` so that a temporary fault-injection burst doesn't eject the pod for longer than you'd want in production.
