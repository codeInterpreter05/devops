# Day 14 — Lab: Load Balancing & Proxies

**Goal:** Stand up nginx as a real L7 load balancer in front of 3 backend instances, and directly observe round-robin distribution, passive health-check failover, and IP-hash sticky sessions — not just read about them.

**Prerequisites:** nginx installed (`sudo apt install nginx` on Ubuntu/Debian, `brew install nginx` on macOS), Python 3 (for disposable backend servers — no app code needed), `curl`. Optional for the stretch challenge: `haproxy` and `openssl`.

---

### Lab 1 — Stand up 3 backend servers

We don't need real application servers to prove out load balancing — three instances of Python's built-in HTTP server, each serving a file that identifies itself, is enough to see exactly which backend answered every request.

1. Create three tiny backend directories, each with an `index.html` that names itself:
   ```bash
   mkdir -p ~/day14-lab/backend1 ~/day14-lab/backend2 ~/day14-lab/backend3
   echo "Backend 1 (port 8081)" > ~/day14-lab/backend1/index.html
   echo "Backend 2 (port 8082)" > ~/day14-lab/backend2/index.html
   echo "Backend 3 (port 8083)" > ~/day14-lab/backend3/index.html
   ```
2. Start all three in the background, one per port:
   ```bash
   (cd ~/day14-lab/backend1 && python3 -m http.server 8081 &)
   (cd ~/day14-lab/backend2 && python3 -m http.server 8082 &)
   (cd ~/day14-lab/backend3 && python3 -m http.server 8083 &)
   ```
3. Confirm each is reachable directly, bypassing any proxy:
   ```bash
   curl http://localhost:8081/
   curl http://localhost:8082/
   curl http://localhost:8083/
   ```

**Success criteria:** Each `curl` returns the distinct text for that backend. You have 3 independent "services" to load-balance across, with no nginx involved yet.

---

### Lab 2 — Configure nginx as an L7 round-robin load balancer

1. Create an nginx config for the lab. On Ubuntu/Debian, drop this in `/etc/nginx/conf.d/day14-lab.conf`; on Homebrew macOS, use `/usr/local/etc/nginx/servers/day14-lab.conf` (or `/opt/homebrew/etc/nginx/servers/` on Apple Silicon):
   ```nginx
   upstream backend_pool {
       server 127.0.0.1:8081;
       server 127.0.0.1:8082;
       server 127.0.0.1:8083;
   }

   server {
       listen 8080;

       location / {
           proxy_pass http://backend_pool;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
       }
   }
   ```
2. Validate the config syntax **before** reloading — this is a habit, not optional:
   ```bash
   sudo nginx -t
   ```
3. Reload nginx to pick up the new config without dropping existing connections:
   ```bash
   sudo nginx -s reload
   ```
4. Hit the load balancer repeatedly and watch the backend rotate:
   ```bash
   for i in $(seq 1 9); do curl -s http://localhost:8080/; done
   ```

**Success criteria:** You see `Backend 1`, `Backend 2`, `Backend 3` cycling in order across the 9 requests — proof nginx is round-robining across the `upstream` pool, not just forwarding to one fixed backend.

---

### Lab 3 — Add passive health checks and test failover

1. Update the `upstream` block to add passive health-check parameters to each server:
   ```nginx
   upstream backend_pool {
       server 127.0.0.1:8081 max_fails=2 fail_timeout=10s;
       server 127.0.0.1:8082 max_fails=2 fail_timeout=10s;
       server 127.0.0.1:8083 max_fails=2 fail_timeout=10s;
   }
   ```
   `sudo nginx -t && sudo nginx -s reload` after every config change from here on — treat that as implicit for every remaining step.
2. Find and kill the backend on port 8082 to simulate a crash:
   ```bash
   lsof -ti:8082 | xargs kill
   ```
3. Immediately hit the LB in a loop and watch what happens:
   ```bash
   for i in $(seq 1 12); do curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080/; done
   ```
4. Now request actual content instead of just the status code:
   ```bash
   for i in $(seq 1 9); do curl -s http://localhost:8080/; echo; done
   ```

Expect the first request or two that would have routed to 8082 to briefly show a slower response or a retry as nginx detects the failure via `proxy_next_upstream`'s default failure conditions (`error timeout`), after which port 8082 is marked unavailable for the `fail_timeout` window (10s) and traffic distributes only across backends 1 and 3.

5. Bring backend 2 back, wait past `fail_timeout`, and confirm it rejoins the rotation:
   ```bash
   (cd ~/day14-lab/backend2 && python3 -m http.server 8082 &)
   sleep 11
   for i in $(seq 1 9); do curl -s http://localhost:8080/; echo; done
   ```

**Success criteria:** You can explain, from having watched it happen, that nginx OSS health checking is *passive* — it only reacts because real requests to 8082 failed, and only after `max_fails` failures within `fail_timeout` does it stop trying that backend, retrying it again once the window expires.

---

### Lab 4 — Sticky sessions with `ip_hash`

1. Add `ip_hash;` to the `upstream` block:
   ```nginx
   upstream backend_pool {
       ip_hash;
       server 127.0.0.1:8081 max_fails=2 fail_timeout=10s;
       server 127.0.0.1:8082 max_fails=2 fail_timeout=10s;
       server 127.0.0.1:8083 max_fails=2 fail_timeout=10s;
   }
   ```
2. Reload, then hit the LB repeatedly:
   ```bash
   for i in $(seq 1 9); do curl -s http://localhost:8080/; echo; done
   ```

**Observe:** every single response now comes from the *same* backend. This is `ip_hash` working exactly as designed — and also its real-world limitation on full display: because you're testing from one machine, nginx sees one client IP for every request, so every request hashes to the same backend. In production this is precisely what happens to any group of users sharing a public IP behind corporate NAT or CGNAT — they all get pinned to one backend regardless of how many distinct people are actually behind that IP.

3. To see the hash mapping shift, remove one server from the pool (comment out the 8083 line), reload, and run the loop again. Notice the backend you land on can change even though your "client identity" (IP) didn't — this is the plain-hash remapping problem consistent hashing is designed to reduce.

**Success criteria:** You can state, from direct observation, why `ip_hash` sticky sessions are fragile behind NAT and when the backend pool changes — not just recite it from the README.

---

### Lab 5 — The full config (the core hands-on activity)

Put it all together into the final config: 3 backends, passive health checks, and sticky sessions in one working file.

```nginx
upstream backend_pool {
    ip_hash;

    server 127.0.0.1:8081 max_fails=2 fail_timeout=10s;
    server 127.0.0.1:8082 max_fails=2 fail_timeout=10s;
    server 127.0.0.1:8083 max_fails=2 fail_timeout=10s;
}

server {
    listen 8080;

    location / {
        proxy_pass http://backend_pool;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_next_upstream error timeout http_502 http_503;
    }
}
```

1. Restore all 3 `server` lines (undo the comment from Lab 4), save, then validate and reload:
   ```bash
   sudo nginx -t
   sudo nginx -s reload
   ```
2. Confirm sticky routing still holds:
   ```bash
   for i in $(seq 1 5); do curl -s http://localhost:8080/; echo; done
   ```
   All 5 responses should show the same backend.
3. Kill that specific backend and confirm nginx fails your "sticky" client over to a different backend rather than just erroring out — sticky sessions and health checking working together:
   ```bash
   lsof -ti:8081 | xargs kill   # or whichever port you were pinned to
   for i in $(seq 1 5); do curl -s http://localhost:8080/; echo; done
   ```

**Success criteria:** You have one nginx config file that simultaneously load-balances across 3 backends, marks a failed backend out of rotation reactively, and pins a given client to one backend under normal conditions — and you've watched all three behaviors happen with your own `curl` output, not just read the directives.

---

### Cleanup

```bash
lsof -ti:8081,8082,8083 | xargs kill 2>/dev/null
sudo rm /etc/nginx/conf.d/day14-lab.conf   # or the servers/ path on macOS
sudo nginx -t && sudo nginx -s reload
rm -rf ~/day14-lab
```

---

### Stretch challenge

Reproduce the same 3-backend setup in HAProxy and compare:

```
defaults
    mode http
    timeout connect 5s
    timeout client  30s
    timeout server  30s

frontend http_in
    bind *:9090
    default_backend backend_pool

backend backend_pool
    balance leastconn
    server backend1 127.0.0.1:8081 check inter 5s rise 2 fall 3
    server backend2 127.0.0.1:8082 check inter 5s rise 2 fall 3
    server backend3 127.0.0.1:8083 check inter 5s rise 2 fall 3
```

Run it with `haproxy -f day14.haproxy.cfg`, enable the stats page (`stats enable; stats uri /haproxy?stats` in a `listen` block bound to another port), and compare it against nginx: notice HAProxy's `check` actively probes all 3 backends continuously (visible immediately on the stats page as UP/DOWN, even with zero real traffic), whereas nginx OSS only marked a backend down *after* you sent it a failing request in Lab 3. Kill a backend and time how long each proxy takes to notice and stop routing to it — that's the practical difference between active and passive health checking.

If you want the TLS angle instead (or as well): generate a self-signed cert (`openssl req -x509 -newkey rsa:2048 -nodes -keyout day14.key -out day14.crt -days 1 -subj "/CN=localhost"`), add a `listen 8443 ssl; ssl_certificate day14.crt; ssl_certificate_key day14.key;` server block in front of the same `backend_pool` upstream, and hit it with `curl -k https://localhost:8443/` — confirming TLS termination at the LB while backends stay on plain HTTP.
