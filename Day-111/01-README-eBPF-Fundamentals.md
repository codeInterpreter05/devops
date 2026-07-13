# Day 111 — Cilium & eBPF: eBPF Fundamentals

**Phase:** 4 – Advanced/Specialization | **Week:** W19 | **Domain:** Service Mesh | **Flag:** ⚡

## Brief

eBPF is arguably the most important low-level Linux technology to hit infrastructure in the last decade — it's the reason Cilium can do what iptables-based `kube-proxy` can't (line-rate performance at high pod/service counts), and it's the engine behind Hubble, Tetragon, and a growing list of observability/security tools (Falco's eBPF mode, Pixie, Parca). This is flagged as interview-critical because "what is eBPF and why does it matter for Kubernetes" is now a standard platform/SRE interview question, and a shallow "it's a fast packet filter" answer doesn't cut it anymore.

This day is split into three files:
1. **This file** — what eBPF is and why it's replacing iptables.
2. **[02-README-Cilium-As-CNI.md](02-README-Cilium-As-CNI.md)** — Cilium as CNI, identity-based network policy, service mesh, bandwidth management.
3. **[03-README-Hubble-Tetragon-And-Cilium-Vs-Istio.md](03-README-Hubble-Tetragon-And-Cilium-Vs-Istio.md)** — Hubble, Tetragon, and how Cilium and Istio relate.

## What eBPF actually is

**eBPF (extended Berkeley Packet Filter)** lets you run small, sandboxed programs *inside the Linux kernel* — triggered by kernel events (a packet arriving, a syscall being made, a function being entered) — without writing a kernel module and without recompiling or rebooting the kernel. This "programmable kernel" capability is what makes it fundamentally different from anything before it in the Linux ecosystem.

Key mechanics:

- **You write** eBPF programs typically in a restricted subset of C (or via higher-level tooling — Cilium's own compiler pipeline, `bpftrace`'s scripting language, or Rust via `aya`), compiled to eBPF bytecode via LLVM/Clang.
- **The kernel verifier** statically analyzes that bytecode before it's allowed to load: no unbounded loops (until recent bounded-loop support), no out-of-bounds memory access, guaranteed termination. This is what makes eBPF *safe* to run in kernel space — a buggy eBPF program is rejected at load time rather than crashing the kernel.
- **JIT compilation** turns the verified bytecode into native machine code for the CPU, so it runs at near-native speed rather than being interpreted.
- **Hook points** — eBPF programs attach to specific kernel hooks: `XDP` (eXpress Data Path — runs at the network driver level, before the kernel even builds a full `sk_buff`, the earliest and fastest point to make a network decision), `tc` (traffic control, on ingress/egress of a network interface), kprobes/uprobes (kernel/user function entry/exit), `cgroup` hooks (socket-level, e.g. `connect()`/`sendmsg()`), and tracepoints.
- **Maps** — eBPF programs share state with each other and with userspace via **eBPF maps** (hash tables, arrays, LRU caches living in kernel memory). This is how Cilium's userspace agent and its in-kernel datapath programs stay in sync (an endpoint-to-identity map, a policy map, a connection-tracking map).

## Why eBPF is replacing iptables for Kubernetes networking

`kube-proxy`'s default mode uses **iptables** (or the newer `ipvs` mode) to implement Service virtual IPs, DNAT, and load balancing across pod endpoints. iptables rules are evaluated as a **linear, sequential list** per packet — every Service and every Endpoint translates into rules/chains that grow with cluster size. The practical consequence: with iptables mode, packet-processing latency for Service routing scales **roughly linearly (O(n)) with the number of Services**, because the kernel walks the rule list top to bottom looking for a match. Clusters with thousands of Services/Endpoints have well-documented iptables-mode latency and `kube-proxy` sync-time problems — a full `iptables-restore` of tens of thousands of rules on every Service/Endpoint change is not instant.

eBPF-based dataplanes (Cilium being the flagship) replace this with **hash-table lookups** in eBPF maps — Service VIP-to-backend resolution becomes an **O(1) map lookup** regardless of how many Services exist. Additionally:

- eBPF programs can act at the `XDP`/`tc` layer, making forwarding/NAT decisions **before** the packet reaches the full Linux networking stack (routing tables, netfilter/conntrack), cutting per-packet overhead significantly.
- Rule updates don't require rewriting a giant sequential rule set — you update a map entry, and every eBPF program consulting that map sees the change immediately and atomically, without the "big iptables-restore" bottleneck.
- Kernel-native connection tracking replaces `nf_conntrack` for many code paths, avoiding a well-known iptables scaling pain point (conntrack table exhaustion under high connection churn).

This is why "eBPF vs iptables for kube-proxy replacement" boils down to: **linear rule evaluation plus full netfilter stack traversal (iptables) vs. constant-time map lookups that short-circuit the stack (eBPF)** — the same problem (Service load balancing), fundamentally different and better-scaling mechanism.

## Points to Remember

- eBPF = sandboxed, verified, JIT-compiled programs run inside the kernel, triggered by events — not "faster iptables," a fundamentally different execution model (programmable kernel vs. static rule matching).
- The kernel verifier is what makes eBPF safe: it rejects programs that could loop forever or access memory out of bounds *before* they ever run.
- `XDP` is the earliest/fastest hook (driver level, before `sk_buff` allocation); `tc` hooks are later (per-interface ingress/egress); kprobes/uprobes/tracepoints instrument arbitrary kernel/userspace functions — different tools pick different hooks depending on what they need to see.
- eBPF maps are the shared-state mechanism between kernel-space programs and userspace agents (like Cilium's agent) — this is how policy/config changes propagate without reloading programs.
- iptables scales roughly linearly with Service/Endpoint count (sequential rule evaluation); eBPF map-based lookups are effectively O(1) — this is the core scaling argument, not just "eBPF is newer so it's faster."

## Common Mistakes

- Describing eBPF as "just a faster packet filter" in an interview — it's a general kernel-programmability mechanism; packet filtering (its BPF ancestor's original use case) is only one of many use cases (tracing, security enforcement, profiling).
- Assuming eBPF programs can do anything arbitrary in the kernel — the verifier's restrictions (bounded loops historically, limited stack size, no arbitrary pointer arithmetic) are real constraints that shape how eBPF programs must be written, often requiring helper functions for anything complex.
- Forgetting that eBPF's benefits depend on kernel version — features like BTF (BPF Type Format, for portable/CO-RE programs), bounded loops, and specific helpers landed in different kernel versions; "just use eBPF" assumes a sufficiently modern kernel, which is a real constraint on older enterprise Linux nodes.
- Conflating `tc`-based and `XDP`-based eBPF programs as interchangeable — `XDP` runs before the kernel builds full packet metadata, so it can't rely on routing/conntrack decisions further up the stack unless the program does that work itself.
