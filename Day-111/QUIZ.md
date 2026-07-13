# Day 111 — Quiz: Cilium & eBPF

Try to answer without looking at your notes. Answers are at the bottom.

1. What does the kernel verifier check before allowing an eBPF program to load, and why does that matter?
2. Name three different eBPF hook points and what each is typically used for.
3. Why does Service routing latency in iptables-mode kube-proxy scale roughly linearly with the number of Services, while an eBPF-based approach doesn't?
4. What is Cilium's "identity," and why is it derived from labels instead of IP addresses?
5. What does `kubeProxyReplacement=true` actually change at runtime, and what must you do to the existing kube-proxy DaemonSet?
6. How does a `CiliumNetworkPolicy` differ from a plain Kubernetes `NetworkPolicy`? Give a concrete example only Cilium's version can express.
7. Where does Hubble's flow data actually come from, and why doesn't it need a separate packet-capture mechanism?
8. How does Tetragon's enforcement (e.g., killing a process on a matched `execve`) differ mechanically from a traditional userspace security agent?
9. Explain the "Cilium vs Istio: different layers" framing in your own words — what's wrong with treating them as competitors?
10. What does the `kubernetes.io/egress-bandwidth` annotation do, and what rate-limiting model does Cilium use to implement it?
11. **Interview question:** What is eBPF and why is it revolutionising Kubernetes networking and security?

---

## Answers

1. It checks that the program has no unbounded loops, doesn't access memory out of bounds, and is guaranteed to terminate. This matters because it's what makes it safe to run untrusted-ish code inside the kernel — a buggy program is rejected at load time rather than crashing or hanging the kernel.
2. `XDP` (network driver level, earliest/fastest point to make a forwarding decision, before the kernel builds a full `sk_buff`), `tc` (traffic control, per-interface ingress/egress, used for policy and shaping after basic packet metadata exists), and kprobes/uprobes (function entry/exit in the kernel or in userspace binaries, used for tracing and security observability). Tracepoints and cgroup hooks (socket-level events like `connect()`) are also valid answers.
3. iptables evaluates rules as a sequential list per packet — every Service/Endpoint adds more rules the kernel has to walk through to find a match, so latency grows with rule count. eBPF-based approaches use hash-table (map) lookups for Service-to-backend resolution, which is effectively constant-time (O(1)) regardless of how many Services exist.
4. Identity is a numeric value derived from a pod's security-relevant Kubernetes labels. It's used instead of IP because pod IPs are ephemeral and get reused by unrelated pods after a restart/reschedule, while labels (and therefore identity and policy exposure) stay stable and meaningful across the pod's lifecycle.
5. It moves Service load balancing (ClusterIP/NodePort/LoadBalancer/ExternalIP translation, endpoint selection, session affinity) entirely into Cilium's eBPF programs. You must remove or disable the existing `kube-proxy` DaemonSet, or the two systems can conflict over Service routing and cause intermittent connectivity bugs.
6. Plain Kubernetes `NetworkPolicy` is L3/L4 only (IPs/ports, namespace/label selectors). `CiliumNetworkPolicy` adds L7-aware rules — for example, allowing only `GET /reviews/*` HTTP requests to a service while blocking `POST` requests on the same port, which a plain `NetworkPolicy` cannot express since it has no concept of HTTP methods or paths.
7. It comes directly from the eBPF programs Cilium already has attached to the network datapath for its normal networking/policy-enforcement work. Because Cilium's own dataplane sees every packet/flow anyway, Hubble just surfaces that existing data — no separate packet-capture agent or sidecar is needed.
8. Tetragon's enforcement decision happens synchronously, in-kernel, at the moment the kprobe fires (e.g., on `sys_execve`) — it can kill the process or block the action before it completes. A traditional userspace agent typically observes the event after the fact (via logs or an audit trail) and reacts afterward, which is both slower and easier to race/bypass.
9. Cilium operates at the network/kernel layer (L3/L4, optionally L7) and is mandatory infrastructure — every cluster needs a CNI, and Cilium's L7/mesh capabilities are a bonus on top of that required role. Istio operates primarily at L7 as an optional add-on that assumes a CNI already exists underneath it. They solve problems at different layers of the stack and are commonly run together (Cilium for CNI/L3-L4, Istio for L7 traffic management) rather than as mutually exclusive choices.
10. It sets a per-pod egress bandwidth cap. Cilium implements it using eBPF at the `tc` egress hook combined with the kernel's EDT (Earliest Departure Time) model, which calculates the earliest time each packet is allowed to leave — a more precise, lower-overhead approach than traditional `tc` qdisc-based shaping (like `tbf`).
11. Strong answer: "eBPF lets you run small, verified, JIT-compiled programs inside the Linux kernel, triggered by events like a packet arriving or a syscall firing, without writing a kernel module. For networking, it's revolutionary because it replaces iptables' sequential rule-list evaluation (which scales roughly linearly with the number of Services/rules) with constant-time hash-map lookups, and it can act at the earliest possible point in the packet's path (XDP) to skip unnecessary stack traversal — that's the whole basis for Cilium's kube-proxy replacement. For security, the same kernel-level vantage point lets tools like Tetragon observe or synchronously block syscalls (like `execve`) in-kernel, which is both faster and harder to bypass than traditional userspace security agents that only see events after the fact."
