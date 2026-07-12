# Day 14 — Load Balancing & Proxies: Fundamentals

**Phase:** 0 – Foundation | **Week:** W2 | **Domain:** Networking | **Flag:** —

## Brief

A single server can only take so much traffic before it falls over, and even if it could handle the load, a single point of failure means one crash takes everything down with it. Load balancing solves both problems: it spreads traffic across multiple backend instances and stops routing to any instance that stops responding. Every production system you'll touch — a web app behind an ALB, a Kubernetes Service routing across pods, an API gateway in front of microservices — is a load balancing problem wearing a different name. Get the fundamentals solid here and the rest (Kubernetes Services, Ingress controllers, cloud LBs) reads as "the same idea, different knobs" instead of new material.

This day is split into three focused files:

1. **This file** — L4 vs L7 load balancing, load-balancing algorithms, health checks, and connection draining.
2. **[02-README-Nginx-Reverse-Proxy-And-LB.md](02-README-Nginx-Reverse-Proxy-And-LB.md)** — nginx as a reverse proxy and load balancer: `proxy_pass`, `upstream`, health checks, sticky sessions.
3. **[03-README-HAProxy-And-SSL-Termination.md](03-README-HAProxy-And-SSL-Termination.md)** — HAProxy's frontend/backend model and SSL/TLS termination vs passthrough.

## L4 vs L7 load balancing

The names come from the OSI model layers each type of load balancer operates at.

**L4 (transport layer)** load balancers make routing decisions using only the TCP/UDP connection info — source/destination IP and port (the "5-tuple": src IP, src port, dst IP, dst port, protocol). They never look inside the payload. A packet arrives, the LB picks a backend based on the connection tuple and a balancing algorithm, and from then on it forwards packets for that connection to the same backend — it doesn't know or care whether the payload is HTTP, a database protocol, or raw binary. Because there's no payload parsing, no TLS termination, and often no full proxying (some L4 LBs forward packets directly rather than terminating and re-establishing the connection), L4 balancing is fast and protocol-agnostic. **AWS NLB**, a plain **IPVS/LVS** setup, or HAProxy running in `mode tcp` are all L4.

**L7 (application layer)** load balancers understand the actual protocol — almost always HTTP/HTTPS. The LB **terminates the client's connection** (it *is* the endpoint as far as the client's TCP/TLS stack is concerned), parses the request (method, path, headers, cookies, host header), decides which backend should handle it, and opens a **separate connection** to that backend to forward the request. This is strictly more expensive than L4: two TCP connections instead of one to proxy, full HTTP parsing, and — if TLS is involved — a full handshake plus a decrypt/re-encrypt cycle. In exchange you get routing decisions L4 can never make: **path-based routing** (`/api/*` to one service, `/static/*` to another), **host-based routing** (multiple domains served off one IP), header- or cookie-based routing (canary releases, A/B tests), request rewriting, and SSL termination in one place. **AWS ALB**, nginx or HAProxy configured to speak HTTP, and any API gateway are L7.

```nginx
# L7 example: nginx making a content-based routing decision
location /api/ {
    proxy_pass http://api_backend;
}
location / {
    proxy_pass http://web_backend;
}
```

An L4 load balancer physically cannot do the above — it has no concept of a URL path, only IPs and ports.

### When you need L4 vs L7

Reach for **L4** when: you need the lowest possible latency and highest raw throughput (financial trading systems, gaming, high-volume non-HTTP protocols), you're load-balancing a non-HTTP protocol the LB doesn't need to understand (raw TCP services, some database proxies, MQTT), or the backend itself must terminate TLS (e.g., mutual TLS where the LB must not see the certificate exchange).

Reach for **L7** when: you need routing decisions based on request content (path/host/header/cookie), you want to centralize SSL termination and certificate management, you need cookie-based sticky sessions tied to application semantics, or you're fronting multiple logically distinct services through one entry point (the common case for any modern web app or microservice architecture). This is also why "when do you need L7?" is the natural interview follow-up to "what's the difference between L4 and L7" — the answer is really "whenever the routing decision depends on something inside the request, not just where it's addressed to."

## Load balancing algorithms

How the LB picks *which* backend gets the next connection/request:

- **Round robin** — cycle through backends in order, one request each. Simple, the default in nginx and HAProxy. Works well when all backends are equally sized and requests are roughly uniform in cost.
- **Weighted round robin** — same as above, but backends get a weight so a bigger instance receives proportionally more traffic (`server 10.0.0.1:8080 weight=3;` gets 3x the requests of an unweighted peer).
- **Least connections** — send the next request to whichever backend currently has the fewest active connections. Better than round robin when request duration varies a lot (round robin can pile up slow requests on one backend while others sit idle).
- **IP hash / consistent hashing** — hash a key (usually client IP, sometimes a cookie value) to deterministically pick the same backend for the same key every time. This is how you get **sticky sessions** without a shared session store: the same client always lands on the same backend, so in-memory session state on that backend "just works." Plain hashing has a downside: adding or removing a backend changes the modulus and reshuffles most of the mappings. **Consistent hashing** (e.g., ketama) minimizes that churn — only the keys that mapped to the added/removed backend move, everyone else keeps the same mapping. This matters a lot for caching layers too, where "which backend" often means "which cache node," and reshuffling everything after a scaling event causes a cache-miss stampede.

## Health checks

A backend that's crashed, hung, or overloaded needs to come out of rotation automatically — that's what health checks are for. There are two fundamentally different mechanisms:

- **Active health checks** — the load balancer proactively probes each backend on a fixed interval (TCP connect, or an HTTP request to a dedicated endpoint like `/health` or `/healthz`), independent of real traffic. If a backend fails N consecutive probes it's marked down; if it passes M consecutive probes again it's marked back up. This detects failures *before* real users hit them, and works even during low-traffic periods when passive checks would have little traffic to observe.
- **Passive health checks** — the load balancer doesn't send extra probe traffic; it infers backend health from real requests failing (connection refused, timeout, 5xx response) within a rolling window, and takes the backend out of rotation for a cooldown period after too many failures. This is cheaper (no extra probe load) and needs no dedicated health endpoint, but it means at least one real user request has to fail before the LB reacts, and during low traffic it can take a long time to notice a dead backend.

**Why you want both where possible:** passive checks alone mean your users are the health check — someone eats a failed request every time. Active checks alone mean you only catch what your health endpoint actually verifies — a health check that just returns `200 OK` without touching the database won't catch "the app is up but its DB connection pool is exhausted." Combining both (proactive detection for the common failure modes, passive as a fast-acting backstop for anything the active check's endpoint doesn't cover) gives you faster, more complete failure detection than either alone. Open-source nginx only gives you passive checks natively (see file 2); HAProxy and nginx Plus give you both.

## Connection draining / graceful shutdown

Health checks handle backends that die unexpectedly. **Connection draining** (AWS calls this "deregistration delay") handles the case where you *intentionally* want to take a healthy backend out of rotation — the standard case being a rolling deployment.

If you just yank a backend out of the pool (kill the process, terminate the instance, remove it from the config and reload with no grace period), every request that was mid-flight to that backend at that instant gets a connection reset or a `502 Bad Gateway`. For a deploy that happens every day, that's a steady trickle of user-facing errors.

Draining fixes this by splitting "stop sending it new work" from "shut it down":

1. Mark the backend as **draining** — the LB immediately stops routing *new* connections/requests to it.
2. Let **existing, in-flight requests finish naturally** — the LB (or the backend itself) waits up to a configured timeout for those to complete.
3. Only after requests finish (or the timeout expires) is the backend actually removed/terminated.

In nginx, this looks like marking a server `down;` in the `upstream` block and reloading (`nginx -s reload`) — nginx's graceful reload keeps old worker processes alive just long enough to finish requests they already accepted, while new connections go to workers running the updated config. HAProxy exposes this more directly via its stats socket (`set server backend/name state drain`) or a soft-stop signal. This is the exact same problem Kubernetes solves with `preStop` hooks, readiness gates, and `terminationGracePeriodSeconds` at the pod level — different mechanism, same principle: stop routing new traffic before you kill the thing serving old traffic.

## Points to Remember

- L4 = transport layer, routes on IP/port only, never inspects payload, fast and protocol-agnostic (AWS NLB, plain TCP LB). L7 = application layer, terminates and re-establishes connections to inspect HTTP content, enabling path/host/cookie-based routing and SSL termination — at a CPU cost (AWS ALB, nginx/HAProxy in HTTP mode).
- You need L7 specifically when the routing decision depends on something *inside* the request (path, host, header, cookie) or you want SSL termination and cert management centralized.
- Round robin (even split) → weighted round robin (uneven backend capacity) → least connections (uneven request duration) → IP hash/consistent hashing (need sticky sessions, and want minimal remapping on scale events).
- Active health checks = LB proactively probes on an interval, independent of traffic; passive = LB infers health from real traffic failures. Use both when possible — active catches problems before users do, passive is a fast, low-overhead backstop for what active doesn't cover.
- Connection draining = stop routing *new* traffic to a backend but let *in-flight* requests finish before removing it — essential for zero-downtime rolling deploys.

## Common Mistakes

- Assuming "load balancer" always means L7 just because that's the common cloud default (ALB) — plenty of real infrastructure (NLBs, LVS, raw TCP proxies) is L4, and conflating the two in an interview answer is an immediate signal of shallow understanding.
- Using plain `ip_hash`/modulus-based hashing for a cache-node pool and being surprised that scaling from 3 to 4 nodes invalidates almost the entire cache — that's exactly the problem consistent hashing exists to solve.
- Relying only on a health check endpoint that returns `200 OK` unconditionally (doesn't check DB/downstream dependencies), then wondering why the LB keeps sending traffic to an instance that's actually broken.
- Doing deploys by just killing the old process/instance without marking it as draining first — this is the single most common cause of the "brief blip of 502s during every deploy" complaint.
- Treating passive health checks (nginx OSS's `max_fails`/`fail_timeout`) as equivalent to real active health checking — they only react after real requests fail, they don't proactively catch anything.
