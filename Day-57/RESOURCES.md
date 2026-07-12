# Day 57 — Resources: Service Mesh Intro

## Primary (assigned)

- **Istio docs — Getting Started** (istio.io/latest/docs/setup/getting-started) — the assigned starting point; walks through installing Istio and deploying Bookinfo, exactly what today's lab does, with the official reference commands.

## Deepen your understanding

- **Istio docs — Traffic Management concepts** (istio.io/latest/docs/concepts/traffic-management) — the canonical explanation of `VirtualService`/`DestinationRule` semantics, subset resolution, and match-rule ordering.
- **Envoy docs — Introduction / xDS overview** (envoyproxy.io) — understanding Envoy's design (and the xDS protocol specifically) makes every mesh you encounter afterward — Istio, AWS App Mesh, Consul Connect — much faster to reason about, since they all drive the same underlying proxy technology.
- **Linkerd docs — Architecture** (linkerd.io/2/reference/architecture) — a clear, deliberately concise architecture doc that highlights exactly what Linkerd chose to leave out relative to Istio, and why.
- **William Morgan (Linkerd creator) — "The Bullshit-Free Guide to Service Mesh"** (buoyant.io blog) — a candid, opinionated piece on what service meshes actually solve versus marketing claims — good for building the "when NOT to use one" instinct this note emphasizes.

## Reference / lookup

- **`istioctl analyze` / `istioctl proxy-config` reference** (istio.io/latest/docs/reference/commands/istioctl) — the exact subcommands for debugging live mesh config, worth bookmarking for whenever a `VirtualService` "isn't working."
- **Istio docs — Fault Injection** (istio.io/latest/docs/tasks/traffic-management/fault-injection) — precise syntax and caveats for `delay`/`abort` fault rules used in Lab 4.

## Practice

- **Istio's own Bookinfo sample app** (bundled with every Istio release under `samples/bookinfo/`) — the standard hands-on playground for traffic management, exactly what today's lab uses; revisit it whenever you want to test a new `VirtualService` idea safely.
- **Linkerd's "Getting Started" interactive guide** (linkerd.io/2/getting-started) — run through it once after Istio to feel the install-complexity difference directly rather than only reading about it.
