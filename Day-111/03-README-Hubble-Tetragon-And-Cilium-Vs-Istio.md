# Day 111 — Cilium & eBPF: Hubble, Tetragon & Cilium vs Istio

**Phase:** 4 – Advanced/Specialization | **Week:** W19 | **Domain:** Service Mesh | **Flag:** ⚡

## Brief

eBPF's kernel-level vantage point isn't just useful for moving packets fast — it's also an extraordinary observability and security enforcement point, because you can see every connection, every DNS query, and every syscall a container makes without instrumenting the application at all. Hubble (network observability) and Tetragon (security/runtime observability and enforcement) are Cilium's answer to "now that we're in the kernel anyway, what else can we tell you." Closing with "Cilium vs Istio: different layers" ties the whole day together conceptually.

## Hubble — network observability built on the same eBPF datapath

Hubble doesn't run separate probes — it reads flow events directly from the eBPF programs Cilium already has attached for networking/policy enforcement, so there's no additional packet capture or sidecar needed.

```bash
# See all flows in/out of a namespace, live
hubble observe --namespace default --follow

# See only denied (policy-dropped) flows — the fastest way to debug "why is this blocked"
hubble observe --verdict DROPPED --follow

# Service map / topology in the UI
cilium hubble ui
```

Because flow data comes from the datapath itself, Hubble sees **L3/L4 flow metadata for every connection** (source/destination identity, ports, verdict — forwarded or dropped, and *why*, i.e. which policy caused the drop) and, where L7 visibility is enabled, HTTP/gRPC/DNS/Kafka request-level detail. This makes "why did this request fail" debugging fast: instead of guessing which `NetworkPolicy` is at fault, `hubble observe --verdict DROPPED` shows you the exact denied flow and the identity mismatch that caused it.

## Tetragon — runtime security observability and enforcement

Where Hubble watches the network, **Tetragon** watches everything else at the kernel level: process execution (`execve`), file access, syscalls, and network-related security events, and can both **observe** and **enforce** (kill a process or block a syscall) in real time, entirely in-kernel — no per-event round trip to userspace required for enforcement decisions, which is what makes it low-overhead enough to run everywhere versus classic auditd/sidecar-based agents.

```yaml
apiVersion: cilium.io/v1alpha1
kind: TracingPolicy
metadata:
  name: block-shell-in-prod
spec:
  kprobes:
  - call: "sys_execve"
    syscall: true
    args:
    - index: 0
      type: "string"
    selectors:
    - matchArgs:
      - index: 0
        operator: "Equal"
        values:
        - "/bin/sh"
        - "/bin/bash"
      matchActions:
      - action: Sigkill
```

This example policy kills any process that tries to `execve` a shell — a common "detect/prevent container breakout attempts" control. Because it's enforced by an in-kernel eBPF program attached to the `sys_execve` kprobe, it can't be bypassed by anything happening purely in userspace inside the container, and it adds negligible latency compared to a userspace agent intercepting the same event after the fact.

This is the practical difference between Tetragon and a traditional security agent: traditional tools mostly *observe after the fact* (read logs, audit trails) or intercept at a higher, slower layer; Tetragon can make an enforcement decision synchronously, in-kernel, before the syscall completes.

## Cilium vs Istio: different layers, not direct competitors

The framing that resolves most "which one do I use" confusion:

- **Cilium operates at L3/L4 (and optionally L7) at the network/kernel layer** — it *is* your CNI, it's mandatory infrastructure (every cluster needs a CNI), and its L7/mesh features are a bonus on top of that mandatory role.
- **Istio operates primarily at L7**, as an **optional add-on layer** on top of whatever CNI you already run — it doesn't replace your CNI, it assumes one exists and adds sidecars for app-level traffic management, richer mTLS policy composition (`AuthorizationPolicy`, `RequestAuthentication` for JWT, etc.), and its mature ecosystem of traffic-shaping CRDs (`VirtualService`, `DestinationRule`).

In practice, many production clusters run **both**: Cilium as the CNI (and kube-proxy replacement) for networking/L3-L4 policy/performance, with Istio layered on top for the L7 traffic management and app-level mTLS/authz features Istio's ecosystem is more mature at. Cilium's own sidecar-less mesh mode is increasingly positioned as a lighter-weight Istio alternative for teams who mainly need L7 visibility/basic policy without wanting the sidecar operational overhead — but Istio's traffic-shaping feature depth (fine-grained canary/fault-injection/retry semantics, covered on Day 110) is generally still ahead as of today.

## Points to Remember

- Hubble reads flow data from Cilium's existing eBPF datapath — no extra capture mechanism — and `hubble observe --verdict DROPPED` is the fastest path to root-causing "why was this blocked."
- Tetragon enforces synchronously in-kernel (e.g., `Sigkill` on a matched `execve`) — fundamentally different from and faster than userspace-agent-based security tooling that reacts after an event is already logged.
- Cilium = network/kernel layer (CNI, mandatory); Istio = application layer (optional add-on assuming a CNI exists) — "different layers" is the correct interview framing, not "which one is better."
- Many real clusters run both together: Cilium for CNI plus L3/L4 performance/policy, Istio for L7 traffic management — they are complementary, not mutually exclusive.
- Cilium's sidecar-less mesh trades some traffic-management feature depth for much lower operational/resource overhead versus a full sidecar mesh.

## Common Mistakes

- Treating Hubble as "just another dashboard" and not realizing it can answer *why* a specific flow was dropped (matched policy, denied identity) — far faster than manually cross-referencing NetworkPolicy YAML against logs.
- Deploying Tetragon tracing policies broadly without testing selector scope first — an overly broad `kprobes` selector with a `Sigkill` action can kill legitimate processes cluster-wide; always dry-run with an observe-only action before adding enforcement.
- Pitching Cilium and Istio as competing choices in an interview ("we chose Cilium over Istio") when the accurate framing is usually layer-based coexistence — this is a common signal an interviewer probes for to check real depth versus buzzword familiarity.
- Assuming Tetragon's in-kernel enforcement has zero performance cost — it's far cheaper than userspace alternatives, but kprobe-heavy policies on very hot syscalls (e.g., tracing every `read()`) can still add measurable overhead; scope tracing policies to what you actually need to observe/enforce.
- Forgetting that Hubble's L7 visibility (HTTP/gRPC/Kafka/DNS parsing) requires L7 policy/visibility to be explicitly enabled (and may involve the per-node Envoy proxy) — plain L3/L4-only Cilium installs won't show HTTP method/path detail in `hubble observe` by default.
