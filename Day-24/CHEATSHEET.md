# Day 24 — Cheatsheet: Networking — Services & Ingress

## Service types

```yaml
spec:
  type: ClusterIP        # default; internal-only virtual IP
  type: NodePort          # opens same port on every node (30000-32767); building block for LoadBalancer
  type: LoadBalancer      # provisions a real cloud LB via cloud-controller-manager; billed resource
  type: ExternalName      # pure DNS CNAME alias; no selector, no Endpoints, no proxying
```

```bash
kubectl expose deployment web --port=80 --target-port=8080 --type=ClusterIP
kubectl create service externalname ext-db --external-name=db.example.com
kubectl get svc
kubectl get endpoints <svc>              # actual live backend pod IPs
kubectl get endpointslices -l kubernetes.io/service-name=<svc>
minikube service <svc> --url             # quick local NodePort access
```

## Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts: ["myapp.example.com"]
      secretName: myapp-tls
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix        # or: Exact, ImplementationSpecific
            backend:
              service: { name: web-svc, port: { number: 80 } }
```

```bash
kubectl get ingress
kubectl describe ingress <name>
kubectl get ingressclass
minikube addons enable ingress          # local ingress-nginx, no cloud LB needed
```

## cert-manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
kubectl get pods -n cert-manager
kubectl get clusterissuer
kubectl describe clusterissuer <name>
kubectl get certificate
kubectl describe certificate <name>
kubectl get challenges                   # in-progress ACME HTTP-01/DNS-01 challenges
```

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata: { name: letsencrypt-prod }
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory   # staging: acme-staging-v02...
    email: you@example.com
    privateKeySecretRef: { name: letsencrypt-prod-key }
    solvers:
      - http01: { ingress: { ingressClassName: nginx } }
```

## CoreDNS

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl get svc -n kube-system kube-dns
kubectl get configmap coredns -n kube-system -o jsonpath='{.data.Corefile}'
kubectl logs -n kube-system -l k8s-app=kube-dns
kubectl exec <pod> -- cat /etc/resolv.conf
```

## DNS name formats

```
<svc>.<ns>.svc.cluster.local              # full Service DNS name
<svc>.<ns>                                  # shorthand
<svc>                                        # same-namespace shorthand
<pod>.<headless-svc>.<ns>.svc.cluster.local  # per-pod DNS (StatefulSet + headless Service)
```

## Debug commands

```bash
kubectl run curl-test --image=curlimages/curl --rm -it --restart=Never -- curl -s http://<svc>
kubectl run dns-test --image=busybox:1.36 --rm -it --restart=Never -- nslookup <svc>.<ns>.svc.cluster.local
iptables -t nat -L KUBE-SERVICES -n | head
ipvsadm -Ln
```
