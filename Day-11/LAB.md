# Day 11 — Lab: Docker Deep Dive II

**Goal:** Build a real 3-service Compose stack (app + redis + postgres), prove to yourself that container-to-container DNS and health-gated startup actually work, watch resource limits get enforced in real time, and deliberately break a dependency to observe failover/restart behavior — then look underneath the CLI at the actual network namespaces involved.

**Prerequisites:** Docker Engine with the Compose v2 plugin (`docker compose version` should print a version — if only `docker-compose` with a hyphen works, you're on the deprecated v1 tool and should upgrade). Docker Desktop (Mac/Windows) or Docker Engine + `docker-compose-plugin` (Linux) both work. `ip netns` steps assume a native Linux host or Linux VM — on Docker Desktop, run those specific commands inside the Docker Desktop VM (`docker run -it --privileged --pid=host justincormack/nsenter1` is a common trick) or treat them as read-only-reference and rely on `docker network inspect` instead.

---

### Lab 1 — Build the stack

1. Create a working directory and the app code:
   ```bash
   mkdir -p ~/day11-lab/app && cd ~/day11-lab
   ```
2. Create `app/main.py`:
   ```python
   import os, time
   import redis
   import psycopg2

   r = redis.Redis(host=os.environ["REDIS_HOST"], port=6379, socket_connect_timeout=2)
   pg_conf = dict(
       host=os.environ["POSTGRES_HOST"],
       dbname=os.environ["POSTGRES_DB"],
       user=os.environ["POSTGRES_USER"],
       password=os.environ["POSTGRES_PASSWORD"],
       connect_timeout=2,
   )

   while True:
       try:
           r.ping()
           redis_status = "UP"
       except Exception as e:
           redis_status = f"DOWN ({type(e).__name__})"

       try:
           conn = psycopg2.connect(**pg_conf)
           conn.close()
           pg_status = "UP"
       except Exception as e:
           pg_status = f"DOWN ({type(e).__name__})"

       print(f"[{time.strftime('%H:%M:%S')}] redis={redis_status} postgres={pg_status}", flush=True)
       time.sleep(3)
   ```
3. Create `compose.yaml`:
   ```yaml
   name: day11-lab

   services:
     app:
       image: python:3.12-alpine
       container_name: day11-app
       working_dir: /app
       volumes:
         - ./app:/app:ro
       command: >
         sh -c "pip install --quiet redis psycopg2-binary && python -u main.py"
       environment:
         REDIS_HOST: redis
         POSTGRES_HOST: postgres
         POSTGRES_DB: appdb
         POSTGRES_USER: appuser
         POSTGRES_PASSWORD: apppass
       networks:
         - backend
       depends_on:
         redis:
           condition: service_healthy
         postgres:
           condition: service_healthy
       deploy:
         resources:
           limits:
             cpus: "0.50"
             memory: 128M
       restart: on-failure

     redis:
       image: redis:7-alpine
       container_name: day11-redis
       networks:
         - backend
       healthcheck:
         test: ["CMD", "redis-cli", "ping"]
         interval: 5s
         timeout: 3s
         retries: 5
       deploy:
         resources:
           limits:
             cpus: "0.50"
             memory: 128M

     postgres:
       image: postgres:16-alpine
       container_name: day11-postgres
       environment:
         POSTGRES_DB: appdb
         POSTGRES_USER: appuser
         POSTGRES_PASSWORD: apppass
       volumes:
         - pgdata:/var/lib/postgresql/data
       networks:
         - backend
       healthcheck:
         test: ["CMD-SHELL", "pg_isready -U appuser -d appdb"]
         interval: 5s
         timeout: 3s
         retries: 5
       deploy:
         resources:
           limits:
             cpus: "1.0"
             memory: 256M

   networks:
     backend:
       driver: bridge

   volumes:
     pgdata:
   ```
4. Bring it up and watch startup order happen live:
   ```bash
   docker compose up -d
   docker compose ps
   docker compose logs -f app
   ```

**Success criteria:** `docker compose ps` shows all three services as `running`/`healthy`, and `app`'s logs show `redis=UP postgres=UP` repeating every ~3 seconds. Note that `app` did not even start attempting connections until both `redis` and `postgres` reported healthy — confirm this by re-running `docker compose up -d` after `docker compose down` and watching `docker compose logs -f postgres` alongside `app`: `app`'s container doesn't appear as started until postgres's healthcheck passes.

---

### Lab 2 — Inspect the network from the top down

1. Inspect the Compose-created network (Compose auto-names it `<project>_backend` or similar):
   ```bash
   docker network ls
   docker network inspect day11-lab_backend
   ```
   Identify: the subnet assigned, and the three containers listed under `"Containers"` with their individual IPs.
2. Confirm DNS resolution works because this is a user-defined network:
   ```bash
   docker exec day11-app sh -c "getent hosts redis"
   docker exec day11-app sh -c "getent hosts postgres"
   docker exec day11-app python3 -c "import socket; print(socket.gethostbyname('redis'))"
   ```
3. Confirm the *embedded DNS server* is what's answering, not `/etc/hosts`:
   ```bash
   docker exec day11-app cat /etc/resolv.conf     # should show nameserver 127.0.0.11
   docker exec day11-app cat /etc/hosts            # notice redis/postgres are NOT statically listed here
   ```

**Success criteria:** You can explain, in your own words, why `redis` and `postgres` resolve by name here but would *not* resolve if these same three containers had been started individually with plain `docker run` (no `--network`) instead of via Compose.

---

### Lab 3 — Drop into the actual network namespace

This is the "inspect network namespaces" part of today's assignment — go one level below `docker network inspect`'s logical view into the kernel-level plumbing. Run on a native Linux host or Linux VM.

1. Get the PID Docker assigned to the `app` container's main process:
   ```bash
   PID=$(docker inspect -f '{{.State.Pid}}' day11-app)
   echo $PID
   ```
2. Expose that container's network namespace to the standard `ip netns` tooling (Docker doesn't put it in the default location by default):
   ```bash
   sudo mkdir -p /var/run/netns
   sudo ln -sfT /proc/$PID/ns/net /var/run/netns/day11-app
   sudo ip netns list
   ```
3. Look at the namespace from the host side, using host tools instead of `docker exec`:
   ```bash
   sudo ip netns exec day11-app ip addr        # eth0 + lo, exactly what docker network inspect implied
   sudo ip netns exec day11-app ip route       # default route via the bridge gateway
   ```
4. Find the veth pair on the host side that connects this namespace to the bridge:
   ```bash
   ip link show | grep veth
   sudo ip netns exec day11-app ethtool -S eth0 | grep peer_ifindex   # requires ethtool; matches a host-side veth ifindex
   brctl show   # or: ip link show type bridge, and bridge link show
   ```

**Success criteria:** You can point at the specific veth interface on the host that corresponds to the `app` container's `eth0`, and explain the two-hop path a packet takes from `app`'s process out to `redis`: process -> container's `eth0` -> veth pair -> host-side veth end -> Linux bridge -> `redis`'s own veth pair -> `redis`'s `eth0`.

---

### Lab 4 — Watch resource limits get enforced with `docker stats`

1. Start a live view:
   ```bash
   docker stats day11-app day11-redis day11-postgres
   ```
   Confirm the `MEM USAGE / LIMIT` column shows the limits from the compose file (128MiB for app/redis, 256MiB for postgres), not the host's total memory.
2. In a second terminal, generate load against Redis to push its CPU usage and watch `CPU %` respond in the `docker stats` view:
   ```bash
   docker exec day11-redis redis-benchmark -q -n 200000
   ```
3. Cross-check with `docker inspect`:
   ```bash
   docker inspect day11-redis --format '{{.HostConfig.Memory}} bytes / {{.HostConfig.NanoCpus}} nanocpus'
   ```

**Success criteria:** You can point to the exact `docker stats` column that reflects a resource limit versus one that reflects real-time usage, and confirm via `docker inspect` that the numbers in the compose file's `deploy.resources.limits` actually made it into the running container's cgroup configuration.

---

### Lab 5 — Simulate failover: kill a dependency and observe behavior

1. With the stack still running and `docker compose logs -f app` open in one terminal, kill Redis in another:
   ```bash
   docker kill day11-redis
   ```
2. Watch the `app` logs — within a few seconds you should see `redis=DOWN (...)` while `postgres=UP` continues uninterrupted, proving the two dependencies fail independently.
3. Check what Compose does with the dead `redis` container:
   ```bash
   docker compose ps
   ```
   Since `redis` has no `restart:` policy set in this compose file, it stays exited — this is deliberate, so you can observe the difference. Now bring it back:
   ```bash
   docker compose up -d redis
   ```
4. Watch `app`'s logs recover to `redis=UP` once the healthcheck passes again — no restart of the `app` container was needed, because the Python script retries the connection on every loop iteration rather than crashing.
5. Now repeat the same kill against Postgres instead, and notice the difference: `postgres=DOWN` appears, and since `app`'s `command` doesn't crash on a failed `psycopg2.connect()` (it's caught in a `try/except`), `app` itself never restarts — only its printed status changes. Contrast this with what would happen if the script raised an uncaught exception on a failed connection: with `restart: on-failure` set on `app`, Docker would restart the `app` container itself repeatedly until Postgres came back, which you can prove by temporarily removing the `try/except` around the Postgres call, rebuilding, and repeating the kill.

**Success criteria:** You can articulate the distinction between three different resilience behaviors observed here: (a) a dependency container dying and staying dead with no `restart:` policy, (b) an application retrying transient connection failures without crashing, and (c) what `restart: on-failure` would do if the application *did* crash — and why `depends_on: condition: service_healthy` only governs initial startup ordering, not what happens if a dependency dies later.

---

### Cleanup

```bash
docker compose down -v
sudo rm -f /var/run/netns/day11-app
```
`-v` removes the named `pgdata` volume too — confirm you're done inspecting before running this, since it deletes the Postgres data directory for good.

---

### Stretch challenge

Pick one:

1. **Round-robin DNS with replicas.** Scale the app tier: `docker compose up -d --scale app=2` (drop `container_name` from the `app` service first, since it must be unique per container to scale). From inside one container on the `backend` network, run `getent hosts app` repeatedly and observe multiple IPs returned for the same name, confirming Docker's embedded DNS resolves a shared service name to multiple containers via round-robin.
2. **Trigger and observe an OOM kill.** Temporarily drop `redis`'s memory limit to something unrealistically low (e.g., `memory: 20M`), `docker compose up -d redis`, then run `docker exec day11-redis redis-benchmark -q -n 500000 -d 100000` to push memory usage past the ceiling. Watch the container die, then run `docker inspect day11-redis --format '{{.State.OOMKilled}}'` to confirm the kernel OOM killer (scoped to Redis's cgroup) is what took it down, not a crash inside Redis itself.
