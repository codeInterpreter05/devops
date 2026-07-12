# Day 11 — Docker Deep Dive II: Docker Networking

**Phase:** 0 – Foundation | **Week:** W2 | **Domain:** Containers | **Flag:** —

## Brief

Day 3-4 territory covered running individual containers; today is about making containers talk to each other and to the outside world in a way that survives more than a toy demo. Docker networking is where most "it worked on my machine" container bugs actually live — a service that can't reach its database, a container that's reachable on `localhost` but not from another container, a compose stack where DNS resolution mysteriously doesn't work. Understanding *why* requires going one level below the `docker network` CLI, into Linux network namespaces and veth pairs, because that's genuinely what Docker is orchestrating under the hood.

This day is split into three focused files:

1. **This file** — network drivers (bridge/host/none/overlay), the namespace/veth mechanics underneath them, container-to-container DNS, and cross-host communication.
2. **[02-README-Volumes-And-Bind-Mounts.md](02-README-Volumes-And-Bind-Mounts.md)** — named volumes vs bind mounts vs tmpfs, and when to use each.
3. **[03-README-Compose-And-Resource-Limits.md](03-README-Compose-And-Resource-Limits.md)** — Compose v2 file structure, `depends_on` health conditions, and cgroup-enforced resource limits.

## The four network drivers

Every container gets a network namespace when it's created. A **network driver** decides what that namespace is connected to and how.

### `bridge` (the default)

When the Docker daemon starts, it creates a virtual Ethernet bridge on the host called `docker0` (visible with `ip link show docker0` or `brctl show`). Any container run without `--network` is attached to this bridge, gets a private IP in a subnet like `172.17.0.0/16`, and can reach the internet via NAT through the host (iptables `MASQUERADE` rules Docker installs automatically).

Two flavors matter in practice:

- **The default bridge network** (literally named `bridge`) — exists automatically, all unqualified containers land here. Containers *can* reach each other by IP, but **not by name** — there is no embedded DNS on this network for historical/compatibility reasons. You'd need the deprecated `--link` flag or manually-managed `/etc/hosts` entries to get name resolution.
- **User-defined bridge networks** (`docker network create mynet`) — functionally similar (still a Linux bridge + NAT), but Docker additionally runs an embedded DNS resolver for containers attached to it. This is the practical reason you almost always create a custom network for a multi-container app instead of relying on the default one.

```bash
docker network create --driver bridge mynet
docker run -d --name web --network mynet nginx
docker run -it --network mynet alpine ping web   # resolves "web" by name — only works because mynet is user-defined
```

Port publishing (`-p 8080:80`) only matters for bridge/none-style isolated networks — it's Docker adding a `DNAT` iptables rule so traffic hitting the host's `8080` gets forwarded into the container's namespace on `80`. Without `-p`, the container is reachable from other containers on the same bridge network, but not from outside the host.

### `host`

```bash
docker run -d --network host nginx
```

The container skips network namespace isolation entirely — it shares the host's network namespace directly. There's no virtual interface, no NAT, no port mapping: if nginx binds to `80` inside the container, it's bound to `80` on the host, full stop. `-p` flags are ignored (and Docker will warn you) because there's no separate namespace to map ports *into*.

Trade-off: you gain raw performance (no NAT/bridge overhead — noticeable for very high packet-rate workloads) and you lose network isolation and the ability to run two containers that both want port `80`. Common in high-throughput proxies/load balancers and some monitoring agents that need to see host-level network interfaces. Not available on Docker Desktop for Mac/Windows in the same way, because the Linux kernel's network namespace concept doesn't map cleanly onto the VM-backed Docker Desktop architecture there — it works on Docker Desktop 4.34+/recent versions with some caveats, but this is a "know it works differently on Linux vs Desktop" gotcha worth remembering.

### `none`

```bash
docker run -it --network none alpine sh
```

The container gets a network namespace with only a loopback interface (`lo`) — no `eth0`, no route to anywhere, not even the host. Used for workloads that should have zero network access as a security boundary (a batch job that only touches mounted files, a sandboxed build step) or as a base you then wire up entirely by hand (rare).

### `overlay`

Bridge networks are host-local — a bridge network's IP range only means something on that one Docker host. `overlay` networks solve the multi-host case: they stitch together container networks across multiple Docker hosts into one flat virtual network, built on **VXLAN** (a tunneling protocol that encapsulates Ethernet frames inside UDP packets, so a container on host A can send a frame to a container on host B as if they were on the same L2 segment, with the actual routing happening over the hosts' real network).

```bash
docker swarm init
docker network create --driver overlay --attachable myoverlay
docker service create --name web --network myoverlay --replicas 3 nginx
```

Overlay networks are a **Docker Swarm** primitive — they require a Swarm to exist (`docker swarm init`) and rely on Swarm's control plane (running on ports `2377/tcp`, `7946/tcp+udp`, `4789/udp` for the VXLAN data path) to distribute network membership and DNS records across nodes. Plain `docker network create --driver overlay` without Swarm active will fail. Kubernetes solves the same cross-host problem differently, with its own CNI plugins (Calico, Cilium, Flannel) — overlay in the Docker-specific sense is a Swarm concept, not a general Docker Engine one.

## What actually implements this: namespaces and veth pairs

The four drivers above are Docker's abstraction over two Linux kernel primitives:

- **Network namespaces** — the kernel can give a process its own isolated view of network interfaces, routing tables, iptables rules, and `/proc/net`. Each container is (by default) one network namespace. `ip netns` is the general Linux tool for this, though Docker doesn't put its namespaces in the default `/var/run/netns` location, so you have to reach them via a symlink trick (see below).
- **veth pairs** — a **virtual Ethernet pair** is two connected virtual NICs that behave like a patch cable: anything sent into one end comes out the other. Docker creates one veth pair per container: one end is placed inside the container's network namespace (renamed `eth0` from the container's point of view), the other end stays in the host's root namespace and is attached to `docker0` (or the user-defined bridge). That's the entire mechanism connecting a container to "the network" — traffic from the container's `eth0` crosses the veth pair to the host side, then gets bridged/NAT'd/forwarded from there.

You can see this yourself:

```bash
docker run -d --name demo nginx
PID=$(docker inspect -f '{{.State.Pid}}' demo)
sudo ln -sfT /proc/$PID/ns/net /var/run/netns/demo   # expose the container's netns to the `ip netns` toolset
sudo ip netns exec demo ip addr                        # see eth0 + lo from inside, using host tooling
sudo ip netns exec demo ip route                       # see its default route (via the bridge)
ip link show | grep veth                               # the host-side veth ends, one per running container
```

This is precisely what the lab's "inspect network namespaces" exercise asks you to do — Docker CLI commands (`docker network inspect`) show you the *logical* picture (which containers are on which network, their assigned IPs); dropping into the namespace with `ip netns exec` shows you the actual kernel-level plumbing.

## Container-to-container DNS

Docker runs an **embedded DNS server** inside the daemon, reachable from every container at `127.0.0.11` (each container's `/etc/resolv.conf` points here). On a **user-defined network** (bridge or overlay), this resolver answers queries for:

- Container names and `--name`/`--hostname` values.
- Compose service names (Compose creates a user-defined network per project automatically, and every service is resolvable by its service name — this is *why* a Compose `app` service can just connect to `redis:6379` and `postgres:5432` with zero extra configuration).
- Network aliases (`--network-alias`), useful for giving a container an additional resolvable name, e.g. so `db` resolves to whichever replica is currently primary.

Two things frequently trip people up:

1. **The default bridge network gets no DNS.** If you `docker run` two containers without specifying `--network`, they land on the default `bridge` network and can only reach each other by IP — `ping othercontainer` fails with `Name does not resolve` even though both containers are technically on the same L2 segment. This is one of the most common "why can't my containers see each other" support questions, and the fix is always "create/use a user-defined network."
2. **DNS round-robins across replicas.** If multiple containers share the same name/alias (e.g., multiple replicas behind a Compose service, or a Swarm service with several tasks), resolving that name returns multiple `A` records and clients round-robin across them at the DNS layer — no separate load balancer needed for simple cases. Swarm services additionally get a **VIP (virtual IP)** by default that internally load-balances via IPVS, with DNS round-robin as an alternative mode (`--endpoint-mode dnsrr`).

## How two containers on *different hosts* actually communicate

This is the crux of the interview question for today, and it's worth being precise about, because "just use overlay networking" is only half the real-world answer:

- **Overlay networks (Swarm)** genuinely let two containers on different hosts talk directly using container IPs/names, as if on one network — VXLAN encapsulates the traffic and Swarm's gossip protocol (serf, over `7946/tcp+udp`) distributes which container lives on which node. This requires Swarm mode to be active; it doesn't happen with plain standalone Docker Engine.
- **Kubernetes** solves the same problem with a CNI plugin (Calico, Cilium, Flannel, etc.) providing a flat pod network across nodes, plus `kube-proxy`/Services for stable virtual IPs — architecturally analogous to overlay+VIP, but not the same codebase or protocol.
- **The far more common production pattern**, especially when hosts aren't in a Swarm/K8s cluster at all: containers on different hosts do **not** share a network at the container level. Instead, each host publishes ports on its own network stack (`-p 8080:80`, or `--network host`), and cross-host communication happens over the **host's real network** — container A's host talks to container B's host over the actual LAN/VPC, then Docker's port-mapping/NAT forwards that into the target container. In production this is almost always fronted by something that does service discovery and load balancing for you — a cloud load balancer, an internal DNS/service registry (Consul, AWS Cloud Map), or a service mesh (Istio, Linkerd) sitting in front of many replicas across many hosts. Bridge networking alone is host-local and never crosses a host boundary; without Swarm/Kubernetes orchestration handling the overlay/CNI layer, "cross-host container communication" really means "normal host-to-host networking, with Docker's NAT/port-publishing on each end."

The precise, interview-ready framing: *two containers on the same host communicate over a virtual bridge connected via veth pairs; two containers on different hosts require either an orchestrator-provided overlay/CNI network (Swarm's VXLAN overlay, or Kubernetes' CNI), or — far more common outside of orchestrated clusters — each host's real network plus published ports and an external load balancer or service mesh, since Docker's bridge driver by itself has no concept of "other hosts."*

## Points to Remember

- Four drivers: `bridge` (default, NAT'd, per-host), `host` (no isolation, no port mapping needed/possible), `none` (loopback only), `overlay` (multi-host, VXLAN, requires Swarm).
- The **default** bridge network has no embedded DNS; **user-defined** bridge/overlay networks do. Always create a custom network for multi-container setups — Compose does this for you automatically.
- Docker's embedded DNS lives at `127.0.0.11` inside every container's resolv.conf; it resolves container names, Compose service names, and network aliases, and round-robins across multiple matching containers.
- The kernel mechanism underneath every driver except `host`/`none` is: one **network namespace** per container + one **veth pair** connecting that namespace to a Linux bridge on the host.
- Cross-host container communication needs an orchestrator (Swarm overlay or Kubernetes CNI) for a true flat network; absent that, it's published ports over the hosts' real network, typically behind a load balancer or service mesh.

## Common Mistakes

- Running multiple containers on the default `bridge` network and expecting `ping servicename` to work — it won't; only user-defined networks get DNS.
- Assuming `--network host` works identically and gives the same isolation guarantees on Docker Desktop (Mac/Windows) as on native Linux — the underlying VM changes the picture.
- Believing `overlay` is something you get "for free" with any multi-host Docker setup — it requires Swarm mode to be initialized; plain `docker network create --driver overlay` fails otherwise.
- Conflating "containers can reach each other by name" (a DNS/network-scope fact) with "the service is highly available" — DNS round-robin is not health-aware by itself; a dead replica's IP can still be returned until it's removed from the network.
- Forgetting that `-p`/published ports are meaningless on `--network host` (there's no separate namespace to map into) and equally meaningless as a way to reach a container from *inside* the same Docker host's bridge network — other containers should use the container's internal port directly, not the published host port.
