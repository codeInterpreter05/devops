# Day 24 — Resources: Networking — Services & Ingress

## Primary (assigned)

- **learnk8s.io — Kubernetes Networking deep dive** — free, the assigned starting point. Excellent illustrated walkthroughs of Services, kube-proxy modes, and Ingress traffic paths.

## Deepen your understanding

- **Kubernetes.io — Service, Ingress, DNS for Services and Pods** (kubernetes.io/docs/concepts/services-networking/) — the authoritative reference for every Service type, Ingress field, and CoreDNS naming convention referenced in this day's notes.
- **ingress-nginx documentation** (kubernetes.github.io/ingress-nginx) — the canonical controller's own docs, including the full annotation reference used to customize routing/TLS/rewrite behavior.
- **cert-manager documentation** (cert-manager.io/docs) — ACME issuer setup, HTTP-01 vs DNS-01 challenge tradeoffs, and troubleshooting `Challenge`/`Certificate` objects stuck in a non-ready state.
- **CoreDNS documentation** (coredns.io/manual) — the plugin reference for understanding and customizing the `Corefile` (e.g., adjusting `ndots` cluster-wide, adding custom zones).

## Reference / lookup

- `kubectl explain ingress.spec` / `kubectl explain service.spec` — always in sync with your cluster's installed API version.
- **Kubernetes API Reference** (kubernetes.io/docs/reference/generated/kubernetes-api) — exact semantics of `pathType`, `externalTrafficPolicy`, and Service/Ingress fields.

## Practice

- **KillerCoda Kubernetes networking scenarios** (killercoda.com/kubernetes) — free browser-based labs specifically covering Services, Ingress, and DNS debugging.
- Let's Encrypt **staging environment** (acme-staging-v02.api.letsencrypt.org) — practice full ACME issuance flows repeatedly without risking production rate limits, exactly as used in this day's lab.
