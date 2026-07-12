# Day 24 — Lab: Networking — Services & Ingress

**Goal:** Stand up every Service type, install a real Ingress Controller, expose a Service through Ingress with TLS via cert-manager + Let's Encrypt (staging), and directly observe CoreDNS resolving names.

**Prerequisites:** minikube running, `kubectl`, `helm` installed. This lab uses minikube's built-in `ingress` addon for simplicity (no real cloud LB needed) and cert-manager's staging issuer (no risk of Let's Encrypt production rate limits).

```bash
minikube start --driver=docker
minikube addons enable ingress
```

---

### Lab 1 — Service types, side by side

1. Deploy a simple app and expose it three ways:
   ```bash
   kubectl create deployment web --image=nginx:1.25 --replicas=2
   kubectl expose deployment web --name=web-clusterip --port=80 --type=ClusterIP
   kubectl expose deployment web --name=web-nodeport --port=80 --type=NodePort
   kubectl get svc web-clusterip web-nodeport
   ```
2. Confirm `ClusterIP` is unreachable from your host but reachable from inside the cluster:
   ```bash
   kubectl run curl-test --image=curlimages/curl --rm -it --restart=Never -- \
     curl -s -o /dev/null -w "%{http_code}\n" http://web-clusterip
   ```
3. Reach the `NodePort` Service via minikube's own IP:
   ```bash
   minikube service web-nodeport --url
   curl -s -o /dev/null -w "%{http_code}\n" "$(minikube service web-nodeport --url)"
   ```
4. Create an `ExternalName` Service pointing at a public hostname and resolve it:
   ```bash
   kubectl create service externalname ext-example --external-name=example.com
   kubectl run dns-test --image=busybox:1.36 --rm -it --restart=Never -- \
     nslookup ext-example.default.svc.cluster.local
   ```

**Success criteria:** You can explain, using your own command output, why `ClusterIP` failed from outside the cluster, why `NodePort` worked via the node's IP, and that `ExternalName` resolved to a CNAME rather than any cluster-internal IP.

---

### Lab 2 — The core hands-on activity: Ingress + TLS via cert-manager

This is the assigned hands-on activity — do it for real using a staging issuer so you don't hit Let's Encrypt production rate limits during practice.

1. Install cert-manager:
   ```bash
   kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
   kubectl get pods -n cert-manager -w    # wait for all 3 pods Running
   ```
2. Create a **staging** `ClusterIssuer` (avoids production rate limits while learning):
   ```bash
   cat <<'EOF' | kubectl apply -f -
   apiVersion: cert-manager.io/v1
   kind: ClusterIssuer
   metadata:
     name: letsencrypt-staging
   spec:
     acme:
       server: https://acme-staging-v02.api.letsencrypt.org/directory
       email: you@example.com
       privateKeySecretRef:
         name: letsencrypt-staging-key
       solvers:
         - http01:
             ingress:
               ingressClassName: nginx
   EOF
   kubectl describe clusterissuer letsencrypt-staging   # confirm Ready: True
   ```
3. Note: real ACME HTTP-01 challenges require a publicly resolvable domain pointed at your Ingress — in a local minikube lab you won't have that, so for this step, **inspect the mechanics without expecting a real cert**:
   ```bash
   cat <<'EOF' | kubectl apply -f -
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: web-ingress
     annotations:
       cert-manager.io/cluster-issuer: letsencrypt-staging
   spec:
     ingressClassName: nginx
     tls:
       - hosts: ["web.example.test"]
         secretName: web-tls
     rules:
       - host: web.example.test
         http:
           paths:
             - path: /
               pathType: Prefix
               backend:
                 service: { name: web-clusterip, port: { number: 80 } }
   EOF
   kubectl describe certificate web-tls
   kubectl get challenges                    # will likely show pending/failed - explain why (no public DNS)
   ```
4. Test routing without TLS by hitting the Ingress with the correct `Host` header, bypassing DNS entirely:
   ```bash
   INGRESS_IP=$(minikube ip)
   curl -s -o /dev/null -w "%{http_code}\n" -H "Host: web.example.test" "http://$INGRESS_IP"
   ```

**Success criteria:** cert-manager and the `ClusterIssuer` are running and `Ready`, you've watched a `Challenge` object get created and understand exactly why it can't complete in a local lab (no public DNS resolving to your Ingress IP), and you've successfully routed a request through the Ingress Controller to your Service using a manually-set `Host` header.

---

### Lab 3 — CoreDNS internals

1. Inspect CoreDNS's own Corefile:
   ```bash
   kubectl get configmap coredns -n kube-system -o jsonpath='{.data.Corefile}'
   ```
2. Check a pod's `resolv.conf` and confirm `ndots:5`:
   ```bash
   kubectl exec web-... -- cat /etc/resolv.conf   # use an actual pod name from `kubectl get pods`
   ```
3. Compare lookup behavior for a short internal name vs. a trailing-dot FQDN:
   ```bash
   kubectl run dns-debug --image=busybox:1.36 --rm -it --restart=Never -- sh -c \
     "time nslookup web-clusterip; time nslookup example.com.; time nslookup example.com"
   ```
4. Scale CoreDNS down to 0 briefly and observe the effect on new lookups (existing cached connections keep working):
   ```bash
   kubectl scale deployment coredns -n kube-system --replicas=0
   kubectl run dns-fail-test --image=busybox:1.36 --rm -it --restart=Never --timeout=10s -- \
     nslookup web-clusterip
   kubectl scale deployment coredns -n kube-system --replicas=2   # restore immediately
   ```

**Success criteria:** You've read the live Corefile, confirmed `ndots:5` in a real pod, observed the timing difference between a bare hostname and a trailing-dot FQDN lookup, and witnessed DNS resolution fail cluster-wide when CoreDNS is scaled to zero (then restored it).

---

### Cleanup

```bash
kubectl delete ingress web-ingress --ignore-not-found
kubectl delete certificate web-tls --ignore-not-found
kubectl delete clusterissuer letsencrypt-staging --ignore-not-found
kubectl delete svc web-clusterip web-nodeport ext-example --ignore-not-found
kubectl delete deployment web --ignore-not-found
kubectl delete namespace cert-manager --ignore-not-found
```

### Stretch challenge

Using `nip.io` (a free wildcard DNS service that resolves `<any-ip>.nip.io` to `<any-ip>` — e.g. `192-168-1-1.nip.io` resolves to `192.168.1.1`), point an Ingress `host` at `<minikube-ip-with-dashes>.nip.io` and attempt a real Let's Encrypt staging HTTP-01 challenge end-to-end (you'll need your minikube IP reachable from the public internet, which typically means running this from a cloud VM rather than a laptop behind NAT). Confirm the `Challenge` object transitions to `valid` and a real (staging, untrusted-by-browsers) certificate gets issued into the `web-tls` Secret.
