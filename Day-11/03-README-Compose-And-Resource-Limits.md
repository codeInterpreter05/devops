# Day 11 — Docker Deep Dive II: Compose v2 and Resource Limits

**Phase:** 0 – Foundation | **Week:** W2 | **Domain:** Containers | **Flag:** —

## Brief

Running one container at a time with `docker run` doesn't scale past a toy example — real applications are multiple services (an app, a cache, a database) that need a shared network, defined startup order, and named volumes wired together consistently. Docker Compose is the declarative answer to that for single-host development and testing. Separately, containers left unbounded can starve or crash the host they run on — resource limits (`--memory`, `--cpus`) are how you cap what a container can consume, enforced by the same **cgroups** mechanism the kernel uses for all process resource accounting.

## Compose v2 vs the legacy v1 tool

- **Compose v1** was a standalone Python tool (`docker-compose`, hyphenated, a separate binary you `pip install`ed). It's deprecated and no longer receiving updates.
- **Compose v2** is a **Docker CLI plugin** written in Go, invoked as `docker compose` (space, no hyphen) — it ships bundled with modern Docker Desktop and Docker Engine installs, integrates directly with the `docker` CLI's plugin system, and is meaningfully faster and more actively maintained. `docker compose version` confirms which you're on; if only the hyphenated `docker-compose` works, you're on the legacy tool and should install the plugin instead.

Command surface is nearly identical between the two (`up`, `down`, `logs`, `ps`, `exec`, `build`), so muscle memory transfers — the distinction mostly matters for installation and knowing which one a given tutorial/CI pipeline is assuming.

## Compose file structure

A `compose.yaml` (or `docker-compose.yml` — both names are recognized; `compose.yaml` is the current preferred name) declares three top-level concepts:

```yaml
services:      # the containers to run — the bulk of the file
  app:
    image: myapp:latest
    depends_on:
      redis:
        condition: service_healthy
    networks:
      - backend
    volumes:
      - app_data:/data

networks:      # custom networks services attach to (Compose also creates a default one automatically)
  backend:
    driver: bridge

volumes:       # named volumes referenced by services
  app_data:
```

Key mechanics worth knowing cold:

- **Compose auto-creates a project-scoped, user-defined bridge network** even if you declare no `networks:` block at all — this is exactly why services in a Compose stack can resolve each other by service name out of the box (see Day 11's networking note): Compose networks are user-defined by construction, so the embedded DNS resolver works automatically.
- **`depends_on` controls startup order, not readiness**, unless you add a health condition. Plain `depends_on: [redis]` only guarantees Docker starts the `redis` container before `app` — it says nothing about whether Redis has finished booting and is actually accepting connections yet. That's a frequent source of "my app crashed on startup because the DB wasn't ready" bugs.
- **`condition: service_healthy`** fixes that: Compose will wait until the depended-on service's **healthcheck** reports healthy before starting the dependent service. This requires the depended-on service to actually define a `healthcheck:` block — `depends_on` with a health condition against a service that has no healthcheck is a config error Compose will refuse to start.

```yaml
services:
  redis:
    image: redis:7
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
  app:
    image: myapp:latest
    depends_on:
      redis:
        condition: service_healthy
```

Other `depends_on` conditions: `service_started` (the default, startup-order only), `service_completed_successfully` (wait for a one-shot init/migration container to exit `0` before starting the next service — useful for a `db-migrate` job that must finish before `app` starts).

## Resource limits

Without limits, a single misbehaving container (a memory leak, a runaway process) can consume all of a host's RAM/CPU and degrade or crash every other container on that host — resource limits are the guardrail.

```bash
docker run -d --memory=512m --memory-swap=512m --cpus=1.5 myapp
```

- **`--memory`** (`-m`) sets the hard RAM ceiling for the container's cgroup. Hit it, and the kernel's OOM killer intervenes (see below) — it does not gracefully throttle memory allocation, it kills a process.
- **`--memory-swap`** sets the combined memory+swap ceiling. Setting it equal to `--memory` (as above) effectively disables swap for the container, forcing a hard cap with no swap cushion — a common production choice, since swapping inside a container is usually a performance cliff rather than a graceful degradation. If unset, it defaults to double `--memory` (i.e., an equal amount of swap on top).
- **`--cpus`** sets a fractional CPU cap (`1.5` = one and a half CPU cores' worth of time) — the container can burst but is throttled once it exceeds this quota within each scheduling period, rather than being killed.
- **`--cpu-shares`** (default `1024`) is a *relative* weight, not a hard cap — it only matters when CPUs are contended: a container with `--cpu-shares=2048` gets roughly twice the CPU time of one at the default `1024` *when the host is under CPU pressure*, but either container can use 100% of a core if nothing else is competing for it. `--cpus` is a ceiling; `--cpu-shares` is a priority weight during contention.

In Compose:

```yaml
services:
  app:
    image: myapp:latest
    deploy:
      resources:
        limits:
          cpus: "1.5"
          memory: 512M
        reservations:
          memory: 256M
```

Note: the `deploy.resources` block is fully honored by `docker compose up` for local single-host use as of Compose v2 (earlier tooling only respected it under `docker stack deploy`/Swarm) — worth confirming against your installed Compose version if limits don't seem to apply, but modern `docker compose` does enforce these on plain `up`.

## How cgroups enforce this, and what OOM looks like inside a container

Every resource limit above is implemented by the kernel's **cgroups** (control groups) subsystem — Docker creates a cgroup per container and writes your `--memory`/`--cpus` values into that cgroup's controller files (conceptually `memory.max`, `cpu.max` under cgroup v2). The kernel then enforces those limits at the process-scheduling and memory-allocation level, the same mechanism used for resource isolation in Kubernetes pods, systemd services, and generally anywhere Linux does resource accounting — Docker isn't inventing new enforcement, just automating cgroup configuration per container.

When a container's memory usage hits its `--memory` ceiling, the kernel's **OOM killer** activates *scoped to that container's cgroup* — it selects and kills a process **inside that cgroup** (typically the container's main process, PID 1 inside the container), not some arbitrary process on the host, and not the host itself. From the outside, `docker inspect <container>` shows `OOMKilled: true` in the container's state, and (absent a restart policy) the container exits. This containment is precisely the point of memory limits: a leaking container dies on its own instead of pressuring the whole host into OOM territory, where the kernel might otherwise have to kill an unrelated, healthy process to free memory.

## `docker stats` — live inspection

```bash
docker stats                          # live-updating table: CPU %, mem usage/limit, net I/O, block I/O, for all running containers
docker stats --no-stream               # one-shot snapshot instead of continuous refresh
docker stats <container> <container2>  # scope to specific containers
```

`docker stats` reads the same cgroup accounting data the kernel uses to enforce limits, so it's the fastest way to confirm a limit is actually being respected (watch `MEM USAGE / LIMIT` approach the ceiling under load) and to catch a container quietly approaching its cap before it gets OOM-killed. `docker inspect <container>` (look at `.HostConfig.Memory`, `.HostConfig.NanoCpus`) confirms exactly what limits were applied to a given container, useful when debugging "why did this get OOM-killed" after the fact.

## Points to Remember

- `docker compose` (v2, Go, CLI plugin) has superseded `docker-compose` (v1, Python, standalone) — same command surface, different install/architecture.
- Compose auto-creates a user-defined network per project, which is why services resolve each other by name with zero extra config.
- `depends_on` alone only orders container *startup*; add `condition: service_healthy` (which requires the target service to define a `healthcheck`) to actually wait for readiness.
- `--memory`/`--memory-swap` are hard caps enforced by cgroups; `--cpus` is a hard CPU ceiling; `--cpu-shares` is a relative priority weight that only bites under contention.
- Hitting a memory limit triggers the kernel OOM killer *scoped to the container's cgroup* — it kills a process inside the container, not the host, and the container shows `OOMKilled: true` in `docker inspect`.
- `docker stats` reads live cgroup accounting data — use it to watch limits in action and catch containers approaching their ceiling before they get killed.

## Common Mistakes

- Writing `depends_on: [db]` and assuming the app won't start until the database is ready to accept connections — it only waits for the `db` container process to *start*, not for the database engine inside it to finish initializing, unless a `service_healthy` condition and matching `healthcheck` are configured.
- Setting `--memory` without `--memory-swap`, then being surprised the container can still balloon past the intended cap — by default swap adds an *additional* amount equal to the memory limit unless swap is explicitly matched or disabled.
- Confusing `--cpus` (a hard ceiling) with `--cpu-shares` (a relative weight) — setting a high `--cpu-shares` value expecting a guaranteed amount of CPU, when it only helps during contention and does nothing if the container is the only thing running.
- Running `docker compose down` when the intent was to also wipe data, or `docker compose down -v` when the intent was to keep it — the `-v` flag is the entire difference between "restart clean" and "delete all named volumes for this stack."
- Not checking `docker inspect` after an unexpected container exit and missing the `OOMKilled: true` field — assuming a crash was an application bug when it was actually a memory limit doing exactly what it was configured to do.
