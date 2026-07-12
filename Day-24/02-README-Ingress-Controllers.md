# Day 24 — Networking: Ingress & Ingress Controllers

**Phase:** 1 – Core DevOps | **Week:** W4 | **Domain:** Kubernetes | **Flag:** ⚡ Interview-critical

## Brief

"Explain how a request travels from the internet to a Pod through an Ingress" is one of the most commonly asked Kubernetes networking interview questions, precisely because the `Ingress` object is one of the most misunderstood pieces of Kubernetes: **the `Ingress` resource itself does nothing** — it's inert config that only becomes meaningful once an **Ingress Controller** reads and acts on it. Getting this split right is the whole battle.

## `Ingress` (the API object) vs. Ingress Controller (the thing that does the work)

`Ingress` is just a Kubernetes object — routing rules stored in etcd like anything else. On its own, creating an `Ingress` with no controller running in the cluster does **absolutely nothing**; there's no traffic path, no listener, nothing. You must separately deploy an **Ingress Controller** — a pod (usually a Deployment, sometimes a DaemonSet) that:

1. Watches the apiserver for `Ingress` objects (and Services/Endpoints they reference).
2. Translates those rules into its own underlying proxy configuration (an Nginx config reload, an Envoy xDS update, a Traefik dynamic config, etc.).
3. Actually terminates and routes the incoming HTTP(S) traffic.

Popular controllers: **ingress-nginx** (the most widely used, community-maintained), **AWS Load Balancer Controller** (translates `Ingress` into an actual AWS ALB, a very different mechanism from ingress-nginx), Traefik, HAProxy, Istio Gateway (mesh-native alternative to `Ingress` entirely via the `Gateway` API).

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx     # tells Kubernetes WHICH controller should handle this Ingress
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-svc
                port: { number: 80 }
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-svc
                port: { number: 80 }
```

`ingressClassName` matters as soon as you have more than one controller in a cluster (common in multi-tenant setups) — without it, a controller might pick up *every* Ingress regardless of intent, or none at all, depending on its configured default-handling behavior.

## The full request path: internet → Pod

1. **DNS** resolves `myapp.example.com` to the IP of a cloud load balancer (an ALB/NLB/ELB, provisioned separately — either manually, or automatically if your Ingress Controller is deployed behind a `LoadBalancer`-type Service).
2. **Cloud load balancer** forwards the raw TCP/TLS connection to one of the Ingress Controller's pods (reached via that controller's own Service, typically `LoadBalancer` type, in front of the controller Deployment).
3. **The Ingress Controller pod** (e.g., an actual Nginx process running inside the ingress-nginx pod) terminates TLS (if configured — see below), inspects the `Host` header and request path, and matches it against the routing rules it has compiled from every `Ingress` object it's watching.
4. Based on the match, it proxies the request to the matched **Service's ClusterIP** (`api-svc` or `web-svc` above) — note it goes through the Service, so kube-proxy/IPVS still does its normal job of picking a backend pod.
5. **kube-proxy's rules** (iptables/IPVS) forward to one specific **Pod IP**, chosen from that Service's current Endpoints.
6. The Pod's container handles the request and responds back along the same path in reverse.

The key insight worth saying explicitly in an interview: **the Ingress Controller is itself just a pod, exposed by its own ordinary `Service` (usually `LoadBalancer` or `NodePort`)** — Ingress doesn't introduce a fundamentally new networking primitive, it's a smart HTTP-aware reverse proxy running as a normal workload, using the same Service/Endpoint machinery as everything else for its "last mile" to your app's pods.

## TLS termination with cert-manager + Let's Encrypt

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: you@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            ingressClassName: nginx
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts: ["myapp.example.com"]
      secretName: myapp-tls          # cert-manager creates/renews this Secret automatically
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: { name: web-svc, port: { number: 80 } }
```

`cert-manager` is a controller that watches for `Ingress` objects annotated with a `ClusterIssuer`/`Issuer`, automatically performs the ACME HTTP-01 (or DNS-01) challenge with Let's Encrypt to prove domain ownership, and writes the resulting certificate + key into the `secretName` Secret referenced in `tls:`. The Ingress Controller then reads that Secret and uses it to terminate TLS. It also handles renewal automatically (Let's Encrypt certs are valid 90 days) — this is the single biggest reason to use cert-manager over manually managing certs: **you stop having expiry-related outages**, which used to be an extremely common self-inflicted incident before cert-manager became standard.

```bash
kubectl get ingress myapp
kubectl describe ingress myapp                       # backend resolution + TLS status
kubectl get certificate                                # cert-manager's Certificate objects, Ready column
kubectl describe certificate myapp-tls                 # ACME challenge status, renewal timing
kubectl get challenges                                  # in-progress ACME challenges, useful when stuck
```

## Points to Remember

- `Ingress` is inert configuration; nothing happens without an Ingress Controller actually running and watching it — this is the single most important thing to say correctly in an interview.
- `ingressClassName` determines which controller handles a given `Ingress` — essential once more than one controller exists in a cluster.
- The Ingress Controller is deployed as an ordinary workload exposed by its own `Service` (typically `LoadBalancer`) — Ingress doesn't bypass the Service/Endpoint model, it sits in front of it and adds HTTP-aware (host/path-based) routing plus TLS termination.
- cert-manager automates ACME certificate issuance and renewal against Let's Encrypt, writing the result into the Secret your Ingress's `tls:` block references — eliminating manual cert rotation as an operational task.
- One shared Ingress + Ingress Controller can front many Services by hostname/path, avoiding the one-`LoadBalancer`-per-Service cost problem from the previous file.

## Common Mistakes

- Creating an `Ingress` object and expecting traffic to flow, having forgotten to (or not realizing you need to) install an Ingress Controller at all — a very common first-timer mistake, since nothing errors, the object just silently does nothing.
- Omitting `ingressClassName` in a cluster with multiple controllers installed, leading to ambiguous or unintended routing (either no controller claims it, or the "default" one grabs everything, including Ingresses meant for a different controller).
- Assuming TLS is "handled by Kubernetes" without cert-manager or manual cert management — a `tls.secretName` that doesn't exist yet, or an expired cert nobody renewed manually, produces a browser TLS error with no clear Kubernetes-side symptom pointing at the cause.
- Debugging Ingress issues by only looking at the `Ingress` object's YAML — the actual routing behavior lives in the controller's generated config (e.g., `kubectl exec` into an ingress-nginx pod and check `/etc/nginx/nginx.conf`, or check controller logs) since annotations/edge cases can behave differently than the YAML alone suggests.
- Confusing the AWS Load Balancer Controller's `Ingress` handling (which provisions a real ALB per Ingress, with ALB-specific annotations) with ingress-nginx's model (a single shared proxy pod fleet behind one LB) — the annotations, cost model, and failure modes are quite different between the two, and copying ingress-nginx annotations onto an AWS ALB Ingress (or vice versa) simply won't work.
