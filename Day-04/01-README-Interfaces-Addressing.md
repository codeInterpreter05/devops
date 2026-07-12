# Day 4 — Networking on Linux: Interfaces & Addressing

**Phase:** 0 – Foundation | **Week:** W1 | **Domain:** Linux | **Flag:** ⚡ Interview-critical

## Brief

Before you can debug "the app can't reach the database" or "the pod has no network," you need to know how a Linux box sees its own network identity: what interfaces exist, what addresses they hold, and what's actually listening on them. This is the layer below DNS and HTTP — get comfortable reading `ip addr` and `ss` output cold, because in an incident you won't have time to look up flags. This day is split into three files:

1. **This file** — interfaces, IP addressing, and inspecting sockets/ports (`ip addr`, `ifconfig`, `ss`, `netstat`).
2. **[02-README-DNS-Name-Resolution.md](02-README-DNS-Name-Resolution.md)** — how names become IPs: `dig`, `nslookup`, `/etc/hosts`, `/etc/resolv.conf`, resolution order.
3. **[03-README-TCP-UDP-Diagnostics.md](03-README-TCP-UDP-Diagnostics.md)** — TCP vs UDP, the three-way handshake, `ping`/`traceroute`, `curl -v`, `netcat`, `tcpdump`.

## `ip addr` vs the deprecated `ifconfig`

`ifconfig` and `netstat` come from the `net-tools` package, which has been unmaintained since the early 2000s. Modern distros (recent Debian/Ubuntu, RHEL 8+, Alpine, most container base images) don't ship it by default anymore. The replacement is `iproute2` — the `ip` and `ss` commands — which is actively maintained and exposes features `net-tools` never learned about (network namespaces, multiple routing tables, modern queueing disciplines).

```bash
ip addr show          # or: ip a — list all interfaces and their addresses
ip addr show eth0      # just one interface
ip -4 addr             # IPv4 only
ip -6 addr             # IPv6 only
ip link show           # interface state (UP/DOWN), MAC address, MTU — no IP info
ip addr add 10.0.0.5/24 dev eth0    # add an address (root required)
ip route show           # the routing table — "where does traffic to X go"
ip route get 8.8.8.8    # which interface/gateway would be used to reach this address
```

Reading `ip addr` output:

```
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```

- `<BROADCAST,MULTICAST,UP,LOWER_UP>` — flags. `UP` means administratively enabled; `LOWER_UP` means the physical/virtual link actually has carrier (cable plugged in / veth peer up). You can have `UP` without `LOWER_UP` — that's an interface that's enabled but has no link, a classic "cable unplugged" or "container network not attached yet" signature.
- `inet 172.17.0.2/16` — the address and its **prefix length** (CIDR notation), which *is* the subnet mask expressed as bit count. `/16` = `255.255.0.0`; `/24` = `255.255.255.0`. `ip addr` never shows a dotted-quad netmask — you're expected to read CIDR fluently.
- `scope global` vs `scope link` — global is routable beyond the local segment; link-local (like IPv6 `fe80::/10` or the rare `169.254.0.0/16` from a failed DHCP) only works on the local link.

`ifconfig` still works if installed and is fine to *read* on legacy boxes, but don't write new tooling or muscle memory around it — its output doesn't include the routing table, doesn't understand network namespaces (which is exactly what container networking is built on), and the project itself is dead upstream. If you SSH into a box and `ifconfig: command not found`, that's not broken — that's a normal, modern, minimal image; reach for `ip addr` instead.

## Ports and sockets: `ss` vs the deprecated `netstat`

A **socket** is the kernel's handle for one end of a network connection, identified by the tuple (protocol, local IP, local port, remote IP, remote port) for connected sockets, or just (protocol, local IP, local port) for a listening socket. `ss` ("socket statistics") reads this directly from the kernel's netlink interface, which is why it's dramatically faster than `netstat` (which historically parsed `/proc/net/tcp` line by line in userspace and does more redundant work).

```bash
ss -tulwn          # TCP + UDP, listening only, numeric ports/addresses, wide output
ss -tulpn          # same, but show the owning Process (needs root/sudo for other users' procs)
ss -tan            # all TCP sockets (not just listening), numeric
ss -o state established '( dport = :443 or sport = :443 )'   # filter by state/port
ss -s              # summary statistics (counts per protocol/state)
```

Common flags: `-t` TCP, `-u` UDP, `-l` listening only, `-n` numeric (skip DNS/service-name lookups — faster and avoids a program hanging if DNS is broken), `-p` show process, `-a` all sockets.

```
State   Recv-Q  Send-Q   Local Address:Port    Peer Address:Port   Process
LISTEN  0       128      0.0.0.0:22             0.0.0.0:*           users:(("sshd",pid=812,fd=3))
ESTAB   0       0        10.0.0.5:44210         93.184.216.34:443   users:(("curl",pid=9021,fd=5))
```

- `0.0.0.0:22` means sshd is bound to **all** interfaces on port 22 — reachable from anywhere the firewall allows. `127.0.0.1:5432` bound to loopback only means it's *not* reachable from outside the box, no matter what the firewall says — a very common "why can't I connect to my database from another machine" root cause.
- `Recv-Q`/`Send-Q` for a `LISTEN` socket: `Recv-Q` is the current backlog of pending connections waiting to be `accept()`ed; `Send-Q` for `LISTEN` is the configured max backlog. A `LISTEN` socket with a consistently non-zero, growing `Recv-Q` means the application isn't accepting connections fast enough — a real symptom to know how to read.
- `netstat -tulpn` is the equivalent old-school invocation if that's all you have installed — same mental model, slower implementation.

## Points to Remember

- `ip`/`ss` (iproute2) are the modern, actively maintained tools; `ifconfig`/`netstat` (net-tools) are deprecated and often absent from minimal/container images by default.
- CIDR notation (`/24`) is how `ip addr` expresses subnet size — know common conversions: `/24` = 256 addresses (254 usable) = `255.255.255.0`; `/16` = `255.255.0.0`; `/32` = a single host.
- `UP` without `LOWER_UP` = interface enabled but no physical/virtual carrier — different failure mode than the interface being down entirely.
- A service bound to `127.0.0.1` is invisible from outside the host regardless of firewall rules; a service bound to `0.0.0.0` is reachable on every interface, subject to firewall.
- `ss -n` / always use numeric flags when scripting or under time pressure — reverse DNS lookups can hang or slow output if resolution is broken, which is often exactly when you're debugging.

## Common Mistakes

- Assuming a container/minimal image is "broken" because `ifconfig` or `netstat` isn't found — it's simply not installed; use `ip addr` / `ss` instead of adding net-tools as a reflex.
- Confusing a listening socket bound to `127.0.0.1` with one bound to `0.0.0.0` — deploying a service, binding it to loopback for "security," then spending an hour debugging why other pods/hosts can't reach it.
- Reading `/24` as "24 addresses" instead of "24 bits of network prefix, 256 addresses total, 254 usable" (network and broadcast address reserved).
- Running `ss`/`netstat` without `-n` in a debugging session and waiting on slow reverse-DNS lookups, or misreading a hostname-resolved column as the raw IP.
- Not checking `ip link` for `LOWER_UP`/carrier state and instead assuming any interface listed by `ip addr` is actually usable for traffic.
