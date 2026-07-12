# Day 50 — Probes, Resources & QoS: Health Probes

**Phase:** 1 – Core DevOps | **Week:** W8 | **Domain:** Kubernetes | **Flag:** ⚡ Interview-critical

## Brief

Kubernetes doesn't actually know if your application is healthy — it only knows what you tell it via probes. Get probes wrong and you get two very different flavors of pain: a pod that's stuck receiving traffic while it's actually broken (misconfigured/missing readiness probe), or a pod that gets killed and restarted in an infinite crash loop because it's just slow to start, not actually dead (misconfigured liveness probe with no startup probe). This is one of the most-asked Kubernetes interview topics precisely because getting it wrong in production causes real outages, and almost everyone has a war story.

This day is split into three files:

1. **This file** — Liveness, Readiness, and Startup probes: what each one actually controls and when to use which.
2. **[02-README-Resource-Requests-Limits-And-QoS.md](02-README-Resource-Requests-Limits-And-QoS.md)** — resource requests/limits and the QoS classes they produce.
3. **[03-README-OOMKilled-LimitRange-ResourceQuota.md](03-README-OOMKilled-LimitRange-ResourceQuota.md)** — OOMKilled prevention, LimitRange, and ResourceQuota.

## The three probe types and what each one actually gates

All three probes are configured the same way (HTTP GET, TCP socket, gRPC, or `exec` a command inside the container) but each one controls a **completely different Kubernetes behavior**:

| Probe | What failure does | What it's for |
|---|---|---|
| **Liveness** | Kubelet **kills and restarts** the container | "Is this process stuck/deadlocked and needs a restart to recover?" |
| **Readiness** | Pod is **removed from the Service's Endpoints** (no traffic routed to it), container keeps running | "Is this instance currently able to serve requests right now?" |
| **Startup** | Kubelet **keeps waiting** (doesn't run liveness/readiness checks yet); if it never succeeds within the configured time budget, the container is killed | "Give slow-starting apps enough time to boot before liveness starts judging them" |

**This is the single most commonly botched distinction in real clusters**: engineers configure only a liveness probe (because "health check" sounds like liveness), and a temporary problem that a **readiness** probe should have handled instead — say, a downstream dependency (database) being briefly unavailable — causes the liveness probe to fail too, and Kubernetes starts **restarting a perfectly healthy process** repeatedly, which does nothing to fix the actual downstream outage and adds restart-storm noise on top of it.

**Correct mental model**: readiness answers "should traffic go here right now" (temporary, can flip on/off freely without consequence). Liveness answers "is this process broken beyond its own ability to recover, such that a restart is the only fix" (should be reserved for true deadlocks/hangs, not downstream dependency issues). A liveness probe that returns unhealthy just because a database call timed out is almost always a bug — the app should mark itself *not ready* in that case instead, keep running, and become ready again once the dependency recovers, with zero restarts.

## Startup probes — why they exist

Before Kubernetes 1.16 (startup probes were added in 1.16, GA in 1.20), the only tool for "give my slow-starting JVM app time to boot" was tuning `initialDelaySeconds` on the liveness probe — a blunt, wasteful hammer. If your app usually starts in 10 seconds but occasionally takes 90 seconds under load, you'd either set `initialDelaySeconds: 90` (meaning liveness checks are pointlessly delayed by 80 seconds on the common case) or leave it low and get killed-and-restarted loops on the slow-start cases (which, ironically, made the slow start slower, since restarting resets the clock).

A **startup probe** solves this cleanly: while it's failing, Kubernetes suppresses liveness *and* readiness checks entirely. Once the startup probe succeeds once, it's never checked again for that container's lifetime, and liveness/readiness take over with their own (typically much shorter) timing.

```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30     # allow up to 30 * periodSeconds to boot
  periodSeconds: 10        # -> 300 seconds (5 min) total budget to start
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  periodSeconds: 10
  failureThreshold: 3      # 3 consecutive failures -> restart, checked only AFTER startup succeeds
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  periodSeconds: 5
  failureThreshold: 3
```

Note the pattern above: **`/healthz` vs `/ready` are deliberately different endpoints**, because "is the process alive" and "can I serve traffic right now" are different questions with different logic — `/healthz` might just check the HTTP server is responding, while `/ready` might check DB connection pool health, cache warm-up state, or whether the app has finished loading a large config/model into memory.

## Probe field tuning — the knobs that actually matter

- **`initialDelaySeconds`** — wait this long after container start before the *first* probe (mostly superseded by startup probes for slow-starting apps, but still useful for genuinely fast apps that just need a couple seconds).
- **`periodSeconds`** — how often to probe (default 10s).
- **`timeoutSeconds`** — how long to wait for a response before counting it as a failure (default 1s — frequently too aggressive for apps under load; a probe endpoint contending for the same thread pool as real traffic can time out under load even though the app is fine).
- **`successThreshold`** — consecutive successes needed to flip from failed→healthy (must be 1 for liveness; can be >1 for readiness to avoid flapping).
- **`failureThreshold`** — consecutive failures needed to flip from healthy→failed (default 3).

**Probe mechanisms**, in order of how much they actually test:
- **`exec`** — runs a command inside the container; exit code 0 = success. Most flexible, but forks a process each time (overhead at scale on the node).
- **`tcpSocket`** — just checks a TCP port accepts a connection. Cheap, but doesn't tell you the app is functionally healthy, only that *something* is listening.
- **`httpGet`** — GET request, any 2xx/3xx = success. The most common choice; lets you build a meaningful `/healthz` endpoint that actually checks internal state.
- **`grpc`** — uses the gRPC health checking protocol (`grpc.health.v1.Health`) directly, for gRPC-native services, avoiding the awkwardness of exposing a separate HTTP port just for probing.

## Points to Remember

- Liveness failure = restart the container. Readiness failure = pull it out of Service traffic, no restart. Startup failure (only after exhausting its budget) = kill the container before liveness/readiness ever run.
- Never make liveness fail because of a downstream dependency issue (DB, cache, upstream API) — that's what readiness is for. A liveness probe should only fail when *this process itself* is unrecoverably stuck.
- Use a startup probe for slow-booting apps instead of inflating `initialDelaySeconds` on the liveness probe — it gives a generous one-time boot budget without slowing down steady-state failure detection.
- `timeoutSeconds` defaults to 1 second — bump it for apps where the health endpoint can contend with real traffic under load, or you'll get false-positive failures precisely when the app is busy (not broken).
- `/healthz` (liveness-style) and `/ready` (readiness-style) should usually be different endpoints doing different checks — one is about process health, the other about current serving capability.

## Common Mistakes

- Configuring only a liveness probe and no readiness probe — new pods start receiving traffic the instant the container process starts, even before it's actually ready to serve (no warm cache, DB pool not yet connected), causing errors during every rollout.
- Making the liveness probe check downstream dependencies (DB, external API), causing cascading restarts across the whole deployment when that dependency has a brief blip — restarting doesn't fix a downstream outage and destroys any in-memory state/connections that were fine.
- Setting `failureThreshold`/`periodSeconds` too aggressively (e.g., fails after 2 missed checks at 2-second intervals) for an app with any GC pauses or occasional slow requests, causing restart storms under normal load spikes.
- Forgetting that during a rolling update, a pod without a readiness probe is considered "ready" the instant it's `Running` — this can send traffic to a pod that hasn't finished initializing, especially in JVM apps with JIT warm-up or Go apps still loading a large in-memory cache.
- Using `initialDelaySeconds` on liveness alone to cover for unpredictable startup time, rather than a startup probe — this either wastes time on fast boots or still causes restart loops when startup occasionally exceeds the fixed delay.
