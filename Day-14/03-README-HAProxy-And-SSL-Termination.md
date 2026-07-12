# Day 14 — Load Balancing & Proxies: HAProxy & SSL Termination

**Phase:** 0 – Foundation | **Week:** W2 | **Domain:** Networking | **Flag:** —

## Brief

HAProxy is a dedicated load balancer/proxy — it doesn't serve static files or run application logic, it exists purely to accept connections and route them, and that focus shows up in its config model, its health-checking, and its stats/observability. This file covers HAProxy's frontend/backend structure, then SSL/TLS termination vs passthrough — a decision that shows up regardless of which proxy you use, and one that has real implications for what your internal network traffic looks like.

## HAProxy's frontend/backend model

```
global
    log /dev/log local0
    maxconn 4096

defaults
    mode http
    timeout connect 5s
    timeout client  30s
    timeout server  30s

frontend http_in
    bind *:80
    acl is_api path_beg /api
    use_backend api_servers if is_api
    default_backend web_servers

backend web_servers
    balance roundrobin
    server web1 10.0.0.1:8080 check
    server web2 10.0.0.2:8080 check

backend api_servers
    balance leastconn
    server api1 10.0.0.3:9090 check
    server api2 10.0.0.4:9090 check
```

- **`global`** — process-wide settings (logging, resource limits, user/chroot for privilege drop).
- **`defaults`** — shared settings inherited by every `frontend`/`backend` unless overridden (mode, timeouts) — keeps individual sections short.
- **`frontend`** — where HAProxy listens (`bind`) and how it decides which backend a request goes to. This is where L7 routing decisions live, expressed as **ACLs**: named boolean conditions (`acl is_api path_beg /api` — true if the request path starts with `/api`) combined with `use_backend <name> if <acl>` / `unless <acl>`. ACLs can match on path, header value, SNI (`req.ssl_sni`), HTTP method, source IP, and more — this is HAProxy's equivalent of nginx's `location` blocks plus conditional logic, but more expressive for complex routing rules.
- **`backend`** — a named pool of servers plus a `balance` algorithm: `roundrobin`, `leastconn`, `source` (hash client source IP — HAProxy's version of `ip_hash`), `uri`, and others.
- **`server ... check`** — the `check` keyword turns on HAProxy's built-in **active health check** for that server: a periodic probe independent of real traffic. By default it's a TCP connect check; add `option httpchk GET /health` at the backend level to make it an HTTP-level check against a specific path and expected response, and tune `inter` (probe interval), `rise` (consecutive successes to mark up), `fall` (consecutive failures to mark down) directly on the server line, e.g. `server web1 10.0.0.1:8080 check inter 5s rise 2 fall 3`. This is a genuine, no-license-required active health check — one of the concrete reasons HAProxy is picked over open-source nginx specifically for its health-checking story.
- **Stats page** — `stats enable` plus `stats uri /haproxy?stats` in a listener gives you a live dashboard: per-backend server state (up/down/draining), current/total connections, response times, queue depth. Nginx's equivalent (`stub_status`) is far more limited on open source; a comparable dashboard there needs nginx Plus.

HAProxy's reputation for very high throughput/low latency comes from being purpose-built: it's an event-driven proxy with no general-purpose web-server responsibilities competing for CPU, it's been aggressively tuned for raw connection handling since the early 2000s, and it's proven at large scale (historically run at places like GitHub, Reddit, and Instagram for exactly this reason). That focus is also why its config model looks different from nginx's — everything is expressed in terms of frontends/backends/ACLs rather than location blocks inside a general web server config.

## SSL/TLS termination vs passthrough

**Termination**: the load balancer holds the TLS certificate and private key, terminates the TLS connection from the client, and forwards **plaintext HTTP** to the backend over the internal network.

```
frontend https_in
    bind *:443 ssl crt /etc/haproxy/certs/example.com.pem
    default_backend web_servers
```

```nginx
server {
    listen 443 ssl;
    ssl_certificate     /etc/nginx/certs/example.com.pem;
    ssl_certificate_key /etc/nginx/certs/example.com.key;
    location / { proxy_pass http://backend_pool; }
}
```

Why terminate at the edge: backends stop paying the CPU cost of the TLS handshake and per-request encryption, certificate management/rotation happens in exactly one place instead of on every service instance, and — critically for L7 — the LB can only make content-based routing decisions (path, host, header) on traffic it can actually read, which means it has to be plaintext to the LB at that point. This is also why SNI (Server Name Indication) is special: it's sent unencrypted during the TLS handshake specifically so a proxy *can* route by hostname before decrypting anything.

**Passthrough**: the load balancer forwards the encrypted TLS bytes untouched; the *backend* holds the certificate and does the actual termination. The LB can still make a routing decision using the SNI hostname (visible in cleartext during the handshake) without decrypting the rest of the traffic — HAProxy does this in `mode tcp` with an SNI-matching ACL (`req.ssl_sni`). You need passthrough when: the LB is not allowed to see plaintext application data at all (strict compliance/e2e-encryption requirements), or you're doing **mutual TLS (mTLS)** where the backend needs to validate a specific client certificate itself — if the LB terminates the connection, the backend never sees the client's certificate and can't do that validation, breaking the trust chain the application depends on.

## The tradeoff: what happens to traffic *after* the edge

Terminating at the edge means everything **behind** the load balancer — LB-to-backend, and typically backend-to-backend — is plaintext by default unless you deliberately do something else. That's fine if you fully trust your internal network segment, but it's a real gap in a multi-tenant cluster, a regulated environment (PCI-DSS, HIPAA), or against a threat model that includes "attacker gets a foothold inside the VPC/cluster."

Two ways this gets addressed in practice:

1. **Re-encrypt internally ("SSL bridging")** — the LB terminates the client's TLS connection *and* opens a new (possibly separate-cert) TLS connection to the backend. More CPU on both ends (two full TLS handshakes instead of one) but no plaintext hop anywhere on the wire.
2. **Service mesh mTLS** — instead of every service (or the edge LB) managing its own certs for internal encryption, a mesh (Istio, Linkerd) injects a sidecar proxy next to every workload that automatically wraps pod-to-pod traffic in mutual TLS, with identities and certificate rotation handled by the mesh's control plane (often via SPIFFE/SPIRE identities) rather than manual cert management. This solves the exact same "should internal traffic be encrypted" question the edge LB solves for north-south traffic, but for east-west (service-to-service) traffic, automatically and per-hop rather than as one big edge decision.

The interview-relevant framing: terminating TLS only at the edge is a "castle-and-moat" model — strong perimeter, implicit trust inside. Defense-in-depth (re-encrypting internally, or a mesh doing it for you) assumes the perimeter can be breached and encrypts internal hops too.

## Points to Remember

- HAProxy config = `global` (process settings) + `defaults` (shared settings) + `frontend` (listen + routing via ACLs) + `backend` (server pool + `balance` algorithm).
- `server ... check` gives you genuine active health checking in open-source/free HAProxy — TCP by default, `option httpchk GET /path` for HTTP-level, tunable via `inter`/`rise`/`fall`. This is a real feature-parity gap versus open-source nginx.
- ACLs (`acl name path_beg /x`, `use_backend Y if name`) are HAProxy's mechanism for L7 routing decisions — more composable than nginx's location-block approach for complex conditional routing.
- SSL termination = LB decrypts, backend gets plaintext (cheap for backend, enables L7 routing, centralizes certs). SSL passthrough = LB forwards encrypted bytes untouched, backend terminates (needed for strict e2e-encryption requirements or mTLS where the backend must see the client cert itself).
- Terminating only at the edge leaves internal traffic plaintext by default — re-encrypt internally ("SSL bridging") or use a service mesh's automatic mTLS if that's not an acceptable risk for your environment.

## Common Mistakes

- Assuming HAProxy's `check` gives you exactly what nginx OSS's `max_fails`/`fail_timeout` gives you — they're different mechanisms; HAProxy's is a genuine independent active probe, nginx OSS's is reactive to live traffic failures.
- Trying to do path-based routing on a TLS-passthrough frontend — you cannot inspect an HTTP path without decrypting first; passthrough routing decisions are limited to what's visible before decryption (essentially just SNI).
- Enabling TLS passthrough "for security" without realizing it also means the load balancer can't do any L7 features at all for that traffic — no path routing, no header injection, no WAF inspection at that hop.
- Assuming that because the edge is HTTPS, the whole request path is secure — forgetting that LB-to-backend traffic is plaintext unless SSL bridging or a mesh is explicitly set up.
- Confusing HAProxy's `balance source` (hash on source IP, sticky-ish, same tradeoffs as nginx's `ip_hash` behind NAT) with true cookie-based application-aware stickiness.
