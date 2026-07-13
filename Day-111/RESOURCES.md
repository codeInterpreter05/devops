# Day 111 — Resources: Cilium & eBPF

## Primary (assigned)

- **Cilium documentation** (docs.cilium.io) plus **isovalent.com labs** (free, browser-based, no cluster required) — the assigned starting point. The labs are especially good because they let you run `hubble observe` and policy experiments without provisioning anything locally first.

## Deepen your understanding

- **"What is eBPF?" (ebpf.io)** — the canonical, vendor-neutral explainer for the underlying technology, independent of Cilium specifically; good for making sure your mental model isn't accidentally Cilium-specific.
- **Liz Rice's "Learning eBPF" (O'Reilly, free chapters available)** — the most commonly recommended book-length treatment; strong on the verifier, JIT, and hook-point mechanics covered in file 1.
- **Cilium's own "eBPF Datapath" architecture docs** — a deep, diagram-heavy explanation of exactly how packets flow through Cilium's eBPF programs, useful once the high-level "why" clicks and you want the "how" in detail.

## Reference / lookup

- **Cilium CLI reference** (docs.cilium.io/en/stable/cmdref) — every `cilium` and `hubble` subcommand used in today's cheatsheet, with full flag documentation.
- **CiliumNetworkPolicy editor/reference** (docs.cilium.io — Network Policy section) — the authoritative schema reference for L3/L4/L7 policy syntax.

## Practice

- **Isovalent's "eBPF & Cilium" free labs** (isovalent.com/labs) — guided, hands-on scenarios covering exactly today's activities (kube-proxy replacement, Hubble, network policy) in a disposable sandboxed environment.
- **Tetragon's official "Getting Started" tutorial** — a short, self-contained walkthrough for writing your first `TracingPolicy`, directly useful for today's stretch challenge.
