# Day 14 — Load Balancing & Proxies: Nginx Reverse Proxy & LB

**Phase:** 0 – Foundation | **Week:** W2 | **Domain:** Networking | **Flag:** —

## Brief

nginx is the tool you'll most often reach for to answer "put something in front of my app that load-balances, and can also terminate TLS or handle routing." This file covers nginx specifically as a reverse proxy and load balancer: the directives, what open-source nginx can and can't do out of the box (health checks and sticky sessions have real feature gaps versus nginx Plus), and the exact config you'll write in today's lab.

## Reverse proxy vs forward proxy

A **forward proxy** sits in front of *clients*, on their behalf — it's configured into the client (browser proxy settings, a corporate egress proxy, a VPN), and the server on the other end typically has no idea a proxy is involved; it just sees a request from the proxy's IP. Forward proxies exist to control, inspect, or anonymize *outbound* traffic.

A **reverse proxy** sits in front of *servers*, on their behalf — the client connects to it thinking it *is* the actual server; it has no idea there's a proxy, let alone multiple backends behind it. This is the nginx use case for load balancing: clients talk to nginx, and nginx decides which of N backends actually handles the request.

```nginx
location / {
    proxy_pass http://backend_pool;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

`proxy_pass` forwards the request to the target; the `proxy_set_header` lines matter because, without them, the backend sees nginx's IP as the client and loses the original `Host` header — most apps need `X-Forwarded-For`/`X-Real-IP` to log the real client IP and `X-Forwarded-Proto` to know if the original request was HTTPS (important if the app redirects based on scheme).

**Gotcha worth knowing cold:** whether the `proxy_pass` target URI has a trailing path affects whether the matched `location` prefix is preserved or stripped. `proxy_pass http://backend;` (no path) forwards the URI as-is including whatever matched the location block; `proxy_pass http://backend/;` (trailing slash) replaces the matched location prefix with `/`. Getting this backwards is a classic "why is my app 404ing on every request" nginx bug.

## The `upstream` block

`upstream` defines a named pool of backend servers that `proxy_pass` can target:

```nginx
upstream backend_pool {
    server 10.0.0.1:8080 weight=3;
    server 10.0.0.2:8080;
    server 10.0.0.3:8080 backup;
    server 10.0.0.4:8080 down;
}
```

- `weight=N` — this server gets N times the traffic of an unweighted (`weight=1`) peer under the round-robin/least_conn algorithms. Use it for heterogeneous backend capacity.
- `backup` — only receives traffic when every non-backup server is unavailable. Useful for a cold-standby pool.
- `down` — manually mark a server out of rotation (e.g., for planned maintenance or as the mechanism behind manual connection draining) without deleting the line.
- No directive at all → round robin, nginx's default.

## Load-balancing directives

```nginx
upstream backend_pool {
    least_conn;                       # least-connections instead of round robin
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
    server 10.0.0.3:8080;
}
```

- `least_conn;` — route to the backend with the fewest active connections.
- `ip_hash;` — hash the client IP to deterministically pick the same backend every time (see sticky sessions below).
- `hash $some_variable consistent;` — generic consistent-hash directive, can key on any nginx variable (a cookie, a header, the request URI) — more flexible than `ip_hash`, and the `consistent` flag uses ketama-style hashing to minimize remapping when the pool changes.
- Weighted entries combine with any of the above — `least_conn` still respects `weight=`.

## Health checks: open source vs nginx Plus

This is a real, frequently-tested feature gap. **Open-source nginx only supports passive health checks**, configured directly on the `server` line inside `upstream`:

```nginx
upstream backend_pool {
    server 10.0.0.1:8080 max_fails=2 fail_timeout=10s;
    server 10.0.0.2:8080 max_fails=2 fail_timeout=10s;
    server 10.0.0.3:8080 max_fails=2 fail_timeout=10s;
}
```

`max_fails=2 fail_timeout=10s` means: if nginx counts 2 failed attempts to this server within a 10-second window, mark it unavailable *for the next 10 seconds*, then try it again. "Failed attempt" is defined by `proxy_next_upstream` (default: `error timeout`; you can add `http_500`, `http_502`, `http_503`, `http_504`, etc. to also treat those responses as failures worth failing over from). Defaults are `max_fails=1 fail_timeout=10s` if you don't set them.

This is entirely **reactive** — nginx only notices a backend is down because a real proxied request to it failed. There is no independent proactive probing loop in open-source nginx. **Proactive/active health checks** (a background timer hitting `/health` regardless of live traffic) require either **nginx Plus's `health_check` directive**, or the third-party `nginx_upstream_check_module` patch (also bundled in Tengine, Alibaba's nginx fork) for open-source builds. In practice, teams on open-source nginx either accept passive-only checking, patch in that module, or put something else in front that does active checking and rewrites nginx's config (Consul + consul-template is the classic pattern) or manages upstream membership dynamically.

## Sticky sessions

Two ways to keep a given client's requests landing on the same backend, both important if the app keeps session state in-process instead of in a shared store (Redis, a database):

- **`ip_hash;`** — built into open-source nginx, no extra modules. Deterministically maps client IP to backend. Downsides worth knowing: it breaks down behind NAT/CGNAT (many distinct users appear to share one IP → they all get shoved onto the same backend, defeating the point of load balancing for that group), and it breaks for clients whose IP changes mid-session (common on mobile networks switching between WiFi/cellular).
- **Cookie-based stickiness** — the "correct" mechanism for HTTP session affinity because it doesn't depend on network topology. `sticky cookie srv_id expires=1h domain=.example.com path=/;` inside an `upstream` block is a first-class feature — but it's **nginx Plus only**. On open-source nginx, the workaround is the generic `hash` directive keyed on a cookie value instead of the client IP: `hash $cookie_jsessionid consistent;` — this hashes on whatever session cookie the app already sets, giving you cookie-based-like stickiness without the Plus license, at the cost of it being a hash (client always maps to *a* backend based on that cookie value) rather than nginx actively setting/reading its own dedicated affinity cookie.

## Points to Remember

- Reverse proxy = in front of servers, on their behalf, client is unaware; forward proxy = in front of clients, on their behalf, server is unaware. Load balancing is a reverse-proxy pattern.
- `proxy_pass` trailing-slash-vs-no-trailing-slash changes whether the matched location prefix is stripped — verify with `nginx -t` and a real request, don't assume.
- `upstream` + `server` lines support `weight=`, `backup`, `down`, `max_fails=`, `fail_timeout=` — know what each does without looking it up.
- Open-source nginx health checking is **passive only** (`max_fails`/`fail_timeout`, reactive to real traffic failures); true active/proactive health checks need nginx Plus or a third-party module.
- `ip_hash` is the only sticky-session mechanism built into open-source nginx out of the box — cookie-based stickiness (`sticky cookie`) is nginx Plus only, though the generic `hash $cookie_x consistent;` directive gets you close on open source.

## Common Mistakes

- Forgetting `proxy_set_header Host $host;` and friends, then debugging why the backend app generates wrong redirect URLs or can't tell requests apart by domain.
- Assuming `nginx_upstream_check_module`-style active health checks exist in stock open-source nginx — they don't; only `max_fails`/`fail_timeout` passive checking does.
- Using `ip_hash` for sticky sessions in an environment with corporate NAT or a lot of mobile clients, then being confused about uneven backend load or users randomly losing session state.
- Not setting `proxy_next_upstream` deliberately — the defaults (`error timeout`) won't fail over on a `500`/`502` from a backend that's technically still connectable but returning garbage, unless you explicitly add those status codes.
- Changing the `upstream` server list (adding/removing a backend) while relying on plain `ip_hash`, not realizing that changes the hash distribution for most existing clients, not just the ones tied to the changed server.
