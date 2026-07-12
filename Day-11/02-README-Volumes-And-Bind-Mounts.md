# Day 11 — Docker Deep Dive II: Volumes and Bind Mounts

**Phase:** 0 – Foundation | **Week:** W2 | **Domain:** Containers | **Flag:** —

## Brief

Containers are meant to be disposable — you should be able to `docker rm` one and lose nothing that matters. That only holds if anything that *does* matter (a database's data, uploaded files, generated certs) lives outside the container's writable layer, in storage that outlives the container. Docker gives you three ways to attach outside storage to a container: **named volumes**, **bind mounts**, and **tmpfs mounts**. Picking the wrong one is a common source of "I restarted the container and lost all my data" incidents and of local-dev setups where code changes don't show up without a rebuild.

## Why the container's own filesystem isn't enough

Every container gets a writable layer on top of its image's read-only layers (a union/overlay filesystem, typically `overlayfs` on Linux). Anything written there is genuinely fast and genuinely local — but it's deleted the moment the container is removed (`docker rm`), and it isn't visible to any other container. For stateless application code that's fine. For a database's data directory, it's not: `docker rm postgres_container` would silently destroy every row you'd stored, since the data directory lived only in that removed container's writable layer. Mounts solve this by pointing a path inside the container at storage that lives independently of the container's lifecycle.

## Named volumes

```bash
docker volume create pgdata
docker run -d --name db -v pgdata:/var/lib/postgresql/data postgres:16
docker volume inspect pgdata
docker volume ls
docker volume rm pgdata          # only works if no container references it
```

A **named volume** is storage that Docker itself creates and manages, living under `/var/lib/docker/volumes/<name>/_data` on the host (Linux; the exact path is an internal implementation detail you shouldn't script against). Characteristics that matter:

- **Docker owns the lifecycle.** Volumes survive `docker rm` of the container that used them, and must be explicitly deleted (`docker volume rm`, or `docker compose down -v` for whole stacks) — this is a *feature*, not a leak, but it does mean orphaned volumes accumulate if you're not deliberate about cleanup (`docker volume prune` removes unreferenced ones).
- **Decoupled from host filesystem layout.** You refer to a volume by name, not by a host path. This makes volumes portable across hosts running the same Compose file/image — nobody needs to agree on where `/data/postgres` lives on disk, Docker manages that.
- **Populated from image content on first use.** If the image has files at the mount target already (e.g., a Postgres image ships default config at that path), Docker copies them into the new empty volume the first time it's mounted, so you don't lose baked-in defaults.
- **Volume drivers** let a "volume" be backed by something other than local disk — a plugin can back a named volume with NFS, an EBS-style block volume, or a cloud object store, transparently to the container (`docker volume create --driver <plugin> ...`). This is how you get network-attached, host-independent storage for containers on cloud infrastructure without changing application code.

## Bind mounts

```bash
docker run -d --name web -v /home/deploy/app:/usr/src/app -p 8080:80 node:20
# equivalent, more explicit long syntax:
docker run -d --name web --mount type=bind,source=/home/deploy/app,target=/usr/src/app node:20
```

A **bind mount** maps an *exact* existing path on the host filesystem into the container. Unlike a named volume, there's no Docker-managed abstraction — you're pointing directly at `/home/deploy/app`, and whatever is there (including nothing, or the wrong thing) is exactly what the container sees. Characteristics:

- **Host-filesystem-dependent.** The path must exist in the form you specify on whatever host runs the container — a Compose file with a bind mount tied to `/Users/you/project` works on your Mac and breaks the moment it's deployed to a Linux server with a different path. This is precisely why bind mounts are a local-dev tool, not a production deployment mechanism.
- **Live two-way sync.** Since it's the literal host directory, edits made on the host (your editor saving a file) appear inside the container instantly, and vice versa — this is what makes "live code reload" work: mount your source tree into the container, run a dev server with hot-reload inside, edit files with your normal host editor.
- **No automatic population.** Unlike named volumes, Docker does not copy the image's existing content into an empty bind-mounted directory — if the host path is empty, the container sees an empty directory at that mount point, potentially shadowing files the image expected to find there.
- **Permission/UID mismatches** are the most common bind-mount headache: files created inside the container (as whatever UID the container process runs as) show up on the host owned by that same numeric UID, which may not correspond to any real user on the host, and vice versa — a container running as UID 1000 writing into a bind mount will produce host files owned by whatever local user happens to be UID 1000 (or nobody, showing as an unresolvable UID).

## tmpfs mounts

```bash
docker run -d --tmpfs /app/cache:size=64m,noexec tomcat
docker run -d --mount type=tmpfs,destination=/app/cache,tmpfs-size=64m tomcat
```

A **tmpfs mount** exists only in the host's memory (RAM/swap) — never written to disk at all. It's wiped the instant the container stops, and doesn't count against the container's writable-layer disk usage. Used for secrets you don't want touching disk (a decrypted credential a process needs transiently), or scratch/cache directories where disk I/O would be a bottleneck and durability is irrelevant. Only available on Linux containers (not Windows containers), and doesn't apply across multiple containers the way a named volume does — it's local to the one container's namespace.

## Choosing the right one

| Use case | Right choice | Why |
|---|---|---|
| Database data directory (Postgres, Redis persistence, MySQL) | Named volume | Survives container replacement, portable across hosts, Docker manages lifecycle |
| Local dev: live-reloading your app's source code | Bind mount | You want host edits to appear instantly inside the running container |
| Config file injected at deploy time from a known host path | Bind mount (or better: a Docker/K8s secret/config mechanism in production) | Explicit host path is exactly the intent, but hardcoding host paths doesn't travel well to other machines/orchestrators |
| Sensitive short-lived data that must never touch disk | tmpfs | RAM-only, gone the instant the container stops |
| Cross-host / cloud-backed persistent storage for containers on any node in a cluster | Named volume with a volume driver (NFS, cloud block/object storage plugin) | Volume becomes portable and reattachable regardless of which physical node the container lands on |

The general production rule: **named volumes for anything that must persist and might need to move between hosts; bind mounts for local development convenience where you control the exact host layout.** Databases in production should almost never use a bind mount — you'd be coupling your data's durability and location to one specific host's directory structure, which fights against how containers get rescheduled onto different hosts by any orchestrator.

## Points to Remember

- A container's own writable layer is ephemeral — it disappears with `docker rm`. Anything that must survive needs a volume, bind mount, or tmpfs.
- Named volumes: Docker-managed, live under `/var/lib/docker/volumes`, portable by name, auto-populated from image content on first use, deleted explicitly (`docker volume rm`, `docker volume prune`, `docker compose down -v`).
- Bind mounts: exact host path, host-dependent, no auto-population, best for local dev (live reload), common source of UID/permission mismatches.
- tmpfs: RAM-only, gone on container stop, for secrets/scratch data that shouldn't touch disk.
- Volume drivers let a named volume be backed by NFS/cloud storage — the mechanism that lets stateful containers move between physical hosts in a cluster without losing data.

## Common Mistakes

- Using a bind mount for a production database's data directory, then having the deployment break (or silently start from empty) the moment it runs on a host where that exact path doesn't exist or isn't populated the same way.
- Forgetting `docker compose down -v` vs plain `docker compose down` — the former deletes named volumes (and their data) too; running it against a stack with real data is a common accidental data-loss moment.
- Assuming an empty bind-mounted host directory will inherit the image's default files at that path the way a fresh named volume does — it won't; bind mounts show you exactly what's on the host, even if that's nothing.
- Not noticing UID mismatches: a container process running as a non-root UID writes files that appear "owned by nobody" on the host bind-mounted directory, breaking host-side tooling that expects normal ownership.
- Treating tmpfs like a volume that persists — restarting the container (not just the process inside it) discards everything in a tmpfs mount.
