# Day 13 — TCP/IP & DNS: OSI Model & TCP Handshake

**Phase:** 0 – Foundation | **Week:** W2 | **Domain:** Networking | **Flag:** ⚡ Interview-critical

## Brief

Every "it works on my laptop but not in the cluster" networking bug — a Service that won't route, a pod that can't reach another namespace, a load balancer health check that flaps — is really a question of "which layer is broken?" You can't answer that without a working mental model of the stack packets actually travel through, and without understanding what a TCP connection actually *is* (three packets exchanged before a single byte of your data moves). This note builds both: the OSI model treated as a diagnostic tool instead of a memorization exercise, and the TCP handshake/teardown mechanics that underpin literally every reliable connection you'll ever debug.

This day is split into three focused files:

1. **This file** — the OSI model applied to real DevOps tooling, TCP's three-way handshake and four-way teardown, and how this maps onto pod-to-pod traffic in Kubernetes.
2. **[02-README-DNS-Resolution.md](02-README-DNS-Resolution.md)** — the full DNS resolution chain, record types, TTL/caching, and CoreDNS.
3. **[03-README-TLS-And-HTTP-Evolution.md](03-README-TLS-And-HTTP-Evolution.md)** — the TLS handshake, X.509 certificates, and HTTP/1.1 vs HTTP/2 vs HTTP/3.

## The OSI model, applied

The 7-layer OSI model is usually taught as trivia to memorize in order. It's far more useful as a **debugging checklist**: when something "doesn't work," you ask which layer is actually failing, because the tool you reach for is different at each one.

| Layer | What it is | The tool/concept you actually touch |
|---|---|---|
| L1 — Physical | Bits on a wire/radio/fiber | Rarely yours to touch directly — cloud provider's physical network, NIC link state (`ethtool`) |
| L2 — Data Link | Frames between devices on the same local segment, addressed by MAC | Switches, ARP (IP→MAC resolution), Ethernet. In Kubernetes: the Linux bridge (`cni0`) connecting pod `veth` pairs on the same node operates here |
| L3 — Network | Packets routed between different networks, addressed by IP | Routers, `ip route`, VPC route tables, BGP (Calico uses this for pod routing). Pod-to-pod traffic **across nodes** is routed at this layer |
| L4 — Transport | End-to-end delivery between processes, addressed by port; TCP (reliable, ordered) or UDP (best-effort) | The TCP handshake below lives here. **This is exactly where kube-proxy and a Network Load Balancer operate** — a Kubernetes `ClusterIP` Service is fundamentally an L4 construct: kube-proxy programs `iptables`/IPVS rules that DNAT a `ClusterIP:port` to a chosen backend `PodIP:port`. It has no idea what HTTP is |
| L5/L6 — Session/Presentation | Session establishment, encoding/encryption | Rarely discussed as distinct layers in practice; TLS is often informally slotted here even though operationally it's easiest to think of as sitting between L4 and L7 |
| L7 — Application | The actual protocol your code speaks | HTTP, gRPC, DNS itself. **This is where an Ingress controller, API gateway, or service-mesh sidecar (Envoy) operates** — they can route by hostname, URL path, or header, which an L4 load balancer structurally cannot do because it never parses the payload |

The single most useful takeaway from this table for a DevOps engineer: **a `ClusterIP` Service is L4 (it forwards connections by IP:port), an `Ingress` is L7 (it routes by HTTP host/path).** This is why an Ingress can do things like path-based routing (`/api` → service A, `/` → service B) that a plain Service load balancer never could — the Service never looks past the TCP header.

## The TCP three-way handshake

TCP is connection-oriented: before either side sends a single byte of application data, both sides run a handshake to agree on **initial sequence numbers (ISNs)** and confirm the path works **in both directions** — not just that a packet arrived, but that a reply can get back.

```
Client                          Server
  | --- SYN  (seq=x)        --> |     client picks a random ISN x
  | <-- SYN-ACK (seq=y,     --- |     server picks its own random ISN y,
  |          ack=x+1)            |     and acknowledges the client's SYN
  | --- ACK  (seq=x+1,      --> |     client acknowledges the server's SYN
  |          ack=y+1)            |
  |                               |
  |    <-- connection ESTABLISHED -->  |
```

Why three packets and not two: TCP is **full-duplex** — the client's byte stream and the server's byte stream are numbered independently — so each side needs its own ISN acknowledged. A two-way handshake would only confirm the server received the client's SYN; it would never prove to the *server* that the client can receive the server's reply, since the server has no way to know its SYN-ACK actually arrived until the client's final ACK does. The third packet is what confirms bidirectional reachability, not just one-way delivery.

ISNs are randomized (not simply starting at 0 or 1) specifically so an attacker can't predict them and blindly inject/spoof packets into an existing or new connection (RFC 6528) — sequential, predictable ISNs were a real, exploited weakness in early TCP stacks.

You can watch this live:
```bash
sudo tcpdump -i any -n 'tcp[tcpflags] & (tcp-syn|tcp-ack) != 0 and host example.com'
```
The first packet with only the `S` flag is the SYN; the reply with `S` and `.` (ACK) set is the SYN-ACK; the third packet with just `.` is the final ACK.

## Connection termination: graceful FIN vs abrupt RST

**Graceful close (4-way):**
```
Initiator                       Peer
  | --- FIN --------------> |    "I'm done sending"
  | <-- ACK --------------- |    peer acknowledges (initiator is now half-closed)
  | <-- FIN --------------- |    peer is also done sending, once ready
  | --- ACK --------------> |    initiator acknowledges — connection fully closed
```
It's four packets (not three) because closing is per-direction: the initiator's FIN only says "I won't send more," but the peer might still have data in flight and closes its own side independently, on its own schedule. In practice the middle ACK and the second FIN are often combined into a single packet if the peer has nothing left to send, which can make the exchange look like only 3 packets on the wire.

**Abrupt close (RST):** no negotiation at all — one side sends a `RST` and the connection is torn down immediately, discarding any unacknowledged data. This happens when: connecting to a port with no listener ("connection refused"), an application crashes and the kernel forcibly closes its sockets, a firewall actively rejects a connection, or one side detects a protocol violation. `curl`/`ssh` reporting "Connection reset by peer" means you hit an RST, not a FIN.

**`TIME_WAIT`:** whichever side sends the *final* ACK of the 4-way close (the "active closer") parks that socket in `TIME_WAIT` for **2×MSL** (Maximum Segment Lifetime — historically ~2 minutes per RFC 793, though Linux hardcodes the actual `TIME_WAIT` duration at 60 seconds regardless of MSL tuning) instead of freeing it immediately. Two reasons: (1) if that final ACK is lost, the peer will retransmit its FIN, and the active closer needs to still be around to ACK it again; (2) it prevents delayed/duplicate packets from this now-dead connection arriving later and being mistaken for part of a *new* connection that happens to reuse the same 4-tuple (src IP:port, dst IP:port).

Why this matters operationally: a server handling a high rate of short-lived connections (e.g. an HTTP/1.0-style client that opens a new TCP connection per request, or a load balancer configured to be the active closer) can accumulate tens of thousands of `TIME_WAIT` sockets, exhausting the ephemeral port range or socket table and causing new connections to fail or `bind()` to return `EADDRINUSE`. Mitigations: `SO_REUSEADDR`, tuning `net.ipv4.tcp_tw_reuse` (safe), widening `net.ipv4.ip_local_port_range`, using persistent/pooled connections to reduce churn, and — where you control both ends — designing the protocol so the *client* is the active closer rather than the high-fanout server. (`net.ipv4.tcp_tw_recycle` used to be suggested too — it's removed from modern kernels because it broke connections behind NAT.)

Check `TIME_WAIT` accumulation with:
```bash
ss -tan state time-wait | wc -l
```

## How this underlies a pod talking cross-namespace

This is the mechanical foundation behind the interview question in `QUIZ.md` — read this, then read that quiz answer for the full walkthrough. In short: when Pod A (namespace `foo`) sends a packet to a Service in namespace `bar`, the packet leaves Pod A's network namespace through its `veth` interface, crosses the node's bridge or routing table (the CNI's job — Flannel/Calico/Cilium), and if the destination is a Service `ClusterIP`, it hits `iptables`/IPVS rules that kube-proxy programmed, which **DNAT** the destination from `ClusterIP:port` to an actual backend `PodIP:port` (possibly on a different node, in which case it's then routed over the underlay/overlay network to that node). The TCP three-way handshake described above happens between the **real pod IPs** after this rewrite — TCP has no concept of "namespace" at all. Kubernetes namespaces are an **API-server/RBAC scoping construct**, not a network isolation boundary: by default, any pod can route to any other pod's IP regardless of namespace, unless a `NetworkPolicy` (enforced by a CNI that supports it, like Calico or Cilium — not all do) explicitly restricts it.

## Points to Remember

- OSI layers are a debugging tool: L2 = same-segment framing (MAC/ARP), L3 = routed IP, L4 = ports + TCP/UDP (kube-proxy/Service/NLB territory), L7 = the actual protocol (Ingress/API gateway/mesh territory).
- A Kubernetes `ClusterIP` Service is an L4 abstraction (DNAT by iptables/IPVS); an `Ingress` is L7 (routes by HTTP host/path). Know which one you need for a given routing problem.
- The TCP handshake is 3 packets specifically because TCP is full-duplex — each direction needs its own sequence number synchronized and its own reachability confirmed.
- ISNs are randomized for security (prevents blind sequence-prediction/spoofing attacks), not for load balancing or any performance reason.
- Graceful close is 4-way (FIN/ACK per direction, since each side closes independently); RST is an unnegotiated abrupt teardown used for refused connections, crashes, or protocol violations.
- `TIME_WAIT` is held by the active closer to catch a lost final ACK and let stray duplicate segments expire — under high connection churn this can exhaust ephemeral ports, not memory.
- Kubernetes namespaces do not isolate network traffic by default — any pod can reach any other pod's IP unless a `NetworkPolicy` says otherwise, and that requires a CNI that actually enforces them.

## Common Mistakes

- Treating "the OSI model" as trivia to recite in order rather than a tool to localize a failure — e.g., diagnosing a "can't reach the Service" issue by checking DNS (L7) before confirming the Service's `iptables` DNAT rules exist at all (L4).
- Assuming a Kubernetes NetworkPolicy exists and is enforced by default — most CNIs (including plain Flannel) silently ignore `NetworkPolicy` objects entirely; traffic isn't blocked, and there's no error, it just isn't isolated.
- Confusing "connection refused" (RST — nothing listening, or an active reject) with a timeout (nothing responding at all, usually a routing/firewall drop) — they point to completely different layers of failure and need different debugging (`ss -tlnp` vs `traceroute`/security-group checks).
- Believing `TIME_WAIT` sockets consume meaningful memory/CPU and trying to "fix" a pileup by killing processes — the actual fix is reuse settings, a wider ephemeral port range, or a protocol/connection-pooling change; the sockets are cheap but the *port* is the exhausted resource.
- Forgetting that ISNs are per-direction and independent — assuming the client's and server's sequence numbers are somehow the "same" counter.
