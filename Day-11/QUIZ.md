# Day 11 — Quiz: Docker Deep Dive II

Try to answer without looking at your notes. Answers are at the bottom.

1. Name the four Docker network drivers and, for each, one sentence on what makes it distinct.
2. Why can two containers on the **default** `bridge` network reach each other by IP but not by name, while two containers on a **user-defined** bridge network can use both?
3. What two Linux kernel primitives does Docker actually use to implement container network isolation and connectivity, and what is each one's job?
4. What IP address does Docker's embedded DNS server listen on inside every container, and what does it resolve?
5. Explain, precisely, why `overlay` networking alone doesn't let two containers on two unrelated (non-Swarm, non-Kubernetes) Docker hosts talk to each other, and what production setups typically use instead.
6. What is the core difference between a named volume and a bind mount, in terms of *where the data lives* and *who manages the mapping*?
7. Why would you choose a bind mount over a named volume for local development, and why is that same choice usually wrong for a production database?
8. In Compose, what's the difference between plain `depends_on: [db]` and `depends_on: { db: { condition: service_healthy } }`? What does the latter require the `db` service to define?
9. What's the practical difference between `--cpus=1.5` and `--cpu-shares=2048`?
10. When a container hits its `--memory` limit, what actually happens — and how would you confirm after the fact that this is what killed it?
11. What's the difference between `docker compose down` and `docker compose down -v`?
12. **Interview question:** Explain how container networking works. How do two containers on different hosts communicate?

---

## Answers

1. **`bridge`** (default) — an isolated, NAT'd network per host built on a virtual Linux bridge (`docker0` or a user-defined bridge); requires `-p` to publish ports to the outside. **`host`** — the container shares the host's network namespace directly; no isolation, no port mapping needed or possible. **`none`** — only a loopback interface, no network access at all. **`overlay`** — spans multiple Docker hosts using VXLAN encapsulation, a Docker Swarm primitive requiring Swarm mode to be active.

2. Docker runs an embedded DNS server (`127.0.0.11`) for every user-defined network, resolving container names, Compose service names, and network aliases. The **default** bridge network predates this feature and never got it added (for backward compatibility) — containers there only get IP connectivity via the shared bridge, with no automatic name resolution unless you manually manage `/etc/hosts` or use the deprecated `--link` flag.

3. **Network namespaces** — give each container its own isolated view of network interfaces, routing tables, and iptables rules. **veth pairs** — a virtual two-ended "patch cable" interface; one end lives inside the container's namespace (as `eth0`), the other end stays in the host's root namespace, attached to the bridge. Together: namespace provides isolation, veth pair provides the actual connection across that isolation boundary.

4. `127.0.0.11`, visible in every container's `/etc/resolv.conf`. It resolves container names, Compose service names, and `--network-alias` values — but only for containers on a **user-defined** network (bridge or overlay), not the default bridge network. It also round-robins across multiple containers sharing the same resolvable name.

5. Docker's `bridge` driver is host-local — its IP range and the embedded DNS server only mean something on that one host; nothing in plain Docker Engine stitches two separate hosts' bridge networks together. `overlay` networks do solve the cross-host case, but only within an active Swarm (VXLAN tunnel + Swarm's gossip-based membership distribution) — `docker network create --driver overlay` fails outside of Swarm mode. Kubernetes solves the equivalent problem with a CNI plugin instead. Absent an orchestrator managing that overlay/CNI layer, cross-host container communication in practice means each host publishing ports on its real network interface, with an external load balancer, service registry, or service mesh routing traffic to the right host/container — not a flat container-level network across hosts.

6. A **named volume** is storage Docker itself creates, tracks, and stores under its own managed location (`/var/lib/docker/volumes/...`); you refer to it by name and Docker decides where it physically lives. A **bind mount** maps an exact, pre-existing host filesystem path directly into the container — there's no Docker-managed abstraction, you specify precisely which host directory is exposed.

7. Bind mounts give instant two-way sync between host edits and the running container — exactly what live code reload during local development needs. In production, a bind mount ties your data (or config) to one specific host's exact directory layout, which fights against how any orchestrator reschedules containers onto different hosts; a named volume (ideally backed by a volume driver for network/cloud storage) can move with the container regardless of which physical node it lands on.

8. Plain `depends_on: [db]` only guarantees Docker starts the `db` container *before* the dependent service — it says nothing about whether the database inside is actually ready to accept connections. `depends_on: { db: { condition: service_healthy } }` makes Compose wait until `db`'s healthcheck reports healthy before starting the dependent service, but this requires `db` to define a `healthcheck:` block in the compose file — without one, the health condition has nothing to wait on and Compose will error out.

9. `--cpus=1.5` is a **hard ceiling** — the container can use up to 1.5 CPU cores' worth of time and is throttled beyond that, regardless of what else is running. `--cpu-shares=2048` is a **relative priority weight** (default is 1024) that only affects scheduling when CPUs are actually contended — with no contention, a container at the default `cpu-shares` can still use 100% of a core; the weight only determines the *split* when multiple containers are competing for the same CPU time.

10. The kernel's OOM killer activates, scoped to that specific container's cgroup — it kills a process inside the container (typically its main/PID 1 process), not anything on the host and not other containers. The container then exits (and restarts only if a `restart:` policy says so). Confirm it afterward with `docker inspect <container> --format '{{.State.OOMKilled}}'`, which reports `true` if this is what happened, distinguishing it from an ordinary application crash.

11. `docker compose down` stops and removes the stack's containers and networks but leaves named volumes (and their data) intact. `docker compose down -v` additionally deletes the named volumes referenced by the compose file — meaning it deletes persistent data like a database's data directory. The `-v` is the entire difference between "clean restart" and "wipe everything."

12. Strong answer: "Every container gets its own network namespace — an isolated view of interfaces and routes — connected to the outside via a veth pair, one end in the container, the other attached to a Linux bridge on the host (`docker0` by default, or a user-defined bridge for Compose/custom setups). On a user-defined network, Docker's embedded DNS server at `127.0.0.11` resolves container/service names to their IPs, so two containers *on the same host* just connect to each other by name over that shared bridge. Two containers on **different hosts** can't use that mechanism directly, because a bridge network is host-local — there's no shared L2 segment across hosts. To make that work, you need either an orchestrator providing a real cross-host network: Docker Swarm's `overlay` driver, which tunnels container traffic between hosts over VXLAN, or Kubernetes' CNI plugins (Calico, Cilium, Flannel), which give pods a flat routable network across nodes. Outside of an orchestrated cluster, the far more common production pattern is that containers on different hosts don't share a network at the container level at all — each host publishes the container's port onto its own network stack, and the hosts talk to each other over the real network (LAN/VPC), typically fronted by a load balancer, service registry, or service mesh that handles discovery and routing between the actual hosts and their published ports." Following up with a concrete example (Swarm's VXLAN overlay, or a K8s Service/Ingress in front of pods across nodes) makes the answer land as experience, not just theory.
