# Day 110 — Resources: Istio Deep Dive

## Primary (assigned)

- **Istio documentation** (istio.io/latest/docs) — free, the assigned starting point. The "Concepts" section (Traffic Management, Security, Observability) maps almost 1:1 onto today's three notes.

## Deepen your understanding

- **Istio Bookinfo sample walkthrough** (istio.io/latest/docs/examples/bookinfo) — the canonical hands-on example referenced throughout today's lab; work through the official traffic-shifting and fault-injection tasks after the lab for more reps.
- **Envoy proxy documentation** (envoyproxy.io) — Istio is a control plane *for* Envoy; understanding Envoy's own xDS APIs and listener/cluster/route model directly explains what `istioctl proxy-config` is showing you.
- **"Istio in Action" (Manning)** — a widely recommended deeper book once the docs feel too shallow, especially strong on traffic management and security chapters.

## Reference / lookup

- **istioctl command reference** (istio.io/latest/docs/reference/commands/istioctl) — the authoritative flag-by-flag reference for every command used in today's cheatsheet.
- **Istio API reference** (istio.io/latest/docs/reference/config) — exact schema for `VirtualService`, `DestinationRule`, `Gateway`, `PeerAuthentication`, and every other CRD.

## Practice

- **Kiali's own demo/tutorial mode** — Kiali ships with guided tours that highlight the mTLS padlock and traffic-graph features live against Bookinfo, directly reinforcing Lab 2 and Lab 4.
- **Istio's "Tasks" documentation section** — a series of short, self-contained hands-on tasks (traffic shifting, request timeouts, circuit breaking, mirroring) that pair well as extra reps after today's lab.
