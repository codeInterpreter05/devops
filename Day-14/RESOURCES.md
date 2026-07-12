# Day 14 — Resources: Load Balancing & Proxies

## Primary (assigned)

- **Nginx Beginner's Guide** (nginx.org/en/docs/beginners_guide.html) + **DigitalOcean nginx tutorials** (digitalocean.com/community — search "nginx load balancing", "nginx reverse proxy") — the assigned starting point for this day. Free, practical, covers `proxy_pass`, `upstream`, and basic load balancing in the exact style used in today's lab.

## Deepen your understanding

- **nginx official docs — `ngx_http_upstream_module`** (nginx.org/en/docs/http/ngx_http_upstream_module.html) — the authoritative reference for every `upstream`/`server` directive covered today (`weight`, `max_fails`, `fail_timeout`, `ip_hash`, `least_conn`, `hash`).
- **HAProxy Configuration Manual** (docs.haproxy.org / the `haproxy-configuration-manual` shipped with every install) — dense but exhaustive; the definitive source for ACLs, `balance` algorithms, and health-check tuning (`inter`/`rise`/`fall`).
- **NGINX Learning Center — "Load Balancing with NGINX and NGINX Plus"** — a well-written article specifically contrasting what's open-source vs Plus-only, which resolves a lot of the confusion this day's material surfaces (passive vs active health checks, `ip_hash` vs `sticky cookie`).
- **AWS docs — "Application Load Balancer vs Network Load Balancer"** — a concrete, production framing of the L4/L7 distinction from a cloud provider's own comparison, useful for translating today's concepts directly into interview-ready cloud terms (ALB = L7, NLB = L4).

## Reference / lookup

- `man nginx` / `nginx -h` and `haproxy -h` — quick on-box flag references.
- **nginx.org full directive index** — searchable list of every directive across all modules; faster than guessing syntax from memory.
- **HAProxy's `errors/` example configs and the shipped `haproxy.cfg.default`** — real starting-point configs to diff against when writing your own.

## Practice

- **Today's lab** (`LAB.md` in this folder) — the fastest way to make these concepts stick is to have actually watched `curl` output change as you kill a backend or add `ip_hash`.
- **A local `docker-compose.yml`** running 3 lightweight web containers behind an nginx or HAProxy container — a natural next step up from the Python `http.server` backends used in today's lab, closer to a real containerized deployment.
- **KodeKloud / browser-based nginx and HAProxy playgrounds** (where available for free) — scenario-driven practice for config changes without needing your own VM.
