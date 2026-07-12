# Day 12 — Container Security: Runtime Hardening

**Phase:** 0 – Foundation | **Week:** W2 | **Domain:** Containers | **Flag:** ⚡ Interview-critical

## Brief

Containers are not VMs — they share the host kernel. A VM gets its own kernel, its own memory space, hardware-enforced isolation via the hypervisor. A container is just a regular Linux process with namespaces (isolated views of PIDs, network, mounts, hostname) and cgroups (resource limits) wrapped around it. That difference is the entire reason container security is its own discipline: every container on a host is one kernel exploit or misconfiguration away from touching every other container and the host itself. Runtime hardening is about shrinking that blast radius *before* anything goes wrong, not detecting it after.

This day is split into three focused files:

1. **This file** — running as non-root, read-only filesystems, Linux capabilities, seccomp profiles.
2. **[02-README-Image-Scanning.md](02-README-Image-Scanning.md)** — Trivy, Grype, Docker Scout, and scanning at multiple pipeline stages.
3. **[03-README-Image-Signing-Cosign.md](03-README-Image-Signing-Cosign.md)** — the supply-chain problem, Cosign, keyless signing, admission-time enforcement.

## Why running as root is dangerous

Most public base images (`node`, `python`, `ubuntu`, `nginx` before you add a `USER` line) default their main process to UID 0 — root — because it's the path of least resistance for the image author: no permission errors installing packages, binding ports, writing files. That default is a liability you inherit unless you explicitly override it.

The reason it matters specifically *because containers share the host kernel*: namespaces give a container its own view of PID 1, its own hostname, its own network stack — but by default they do **not** remap UIDs. Root (UID 0) inside a container is, from the kernel's point of view, the same UID 0 the kernel enforces permission checks against everywhere else, gated only by the capability set attached to that process (see below) and whatever namespace boundaries hold. If a container process manages to step outside those boundaries — a kernel privilege-escalation CVE, a container run with `--privileged`, a host path or the Docker socket (`/var/run/docker.sock`) mistakenly mounted in, a misconfigured volume mount of `/` — root inside the container becomes root on the host node. That's a full host compromise, and from there, every other container scheduled on that node is reachable too.

Docker's default capability set already strips the most dangerous capabilities (no `CAP_SYS_ADMIN`, `CAP_SYS_MODULE`, `CAP_SYS_PTRACE` by default), so "root in a default Docker container" is meaningfully less dangerous than "root on a bare host" — but it's still far more privileged than it needs to be for the overwhelming majority of application workloads, which need zero root privileges at all once they're past their initial setup steps (binding a port, writing a log file). Even without any escape, running the app itself as root inside the container means a plain application-level RCE (a deserialization bug, an injected shell command, a vulnerable dependency) hands the attacker root over everything the container can reach — the entire filesystem, every mounted secret, every other process in that container's PID namespace.

`--userns-remap` (Docker daemon-level user namespace remapping) can map container UID 0 to an unprivileged host UID, closing part of this gap even for processes that insist on running as root inside — but it's not the default in most standard Docker or Kubernetes setups, has compatibility friction with some volume/permission workflows, and should be treated as defense-in-depth on top of not running as root in the first place, not a substitute for it.

## `USER` — don't run as root inside the container

```dockerfile
FROM node:20-slim

RUN groupadd --gid 1000 appgroup \
 && useradd --uid 1000 --gid appgroup --shell /usr/sbin/nologin --create-home appuser

WORKDIR /app
COPY --chown=appuser:appgroup . .

USER appuser
CMD ["node", "server.js"]
```

- `USER` must come *before* `CMD`/`ENTRYPOINT` — everything that still needs root (installing OS packages, `chown`ing files, binding to a privileged port `<1024`) has to happen in earlier `RUN` steps, before the switch.
- Prefer a fixed numeric UID (`1000`) over a named user when you also need Kubernetes `runAsUser` to line up, or when the base image doesn't guarantee the same UID→name mapping across versions.
- `docker run --user 1000:1000 myimage` can force this at run time even against an image that defaults to root — useful as a stopgap or a policy-enforced default, but it's better to bake `USER` into the image itself so a forgotten `--user` flag doesn't silently fall back to root.
- Kubernetes equivalent: `securityContext.runAsUser: 1000` plus `runAsNonRoot: true` on the pod or container spec. `runAsNonRoot: true` is the important one — it makes the kubelet refuse to *start* the container if the image's effective user resolves to UID 0, instead of silently running as root because someone forgot the Dockerfile's `USER` line.

## Read-only root filesystems

```bash
docker run --read-only --tmpfs /tmp --tmpfs /var/run myapp:latest
```

The container's root filesystem is mounted read-only; any path the process genuinely needs to write to gets an explicit `--tmpfs` mount (backed by RAM, wiped when the container stops — never used for anything that needs to persist).

This forces an honest inventory of what your app actually writes to at runtime — PID files, unix sockets, cache directories, temp files during upload processing. A well-behaved twelve-factor app writes logs to stdout/stderr rather than a file, which means it needs *no* writable log directory at all. Most stateless services end up needing only `/tmp` and maybe a socket directory.

The security payoff: even a fully successful RCE against the app can't drop a webshell, modify an installed binary, patch application code to add a backdoor, or write a cron persistence mechanism — there's nowhere on disk left to write outside the declared tmpfs mounts.

Kubernetes equivalent: `securityContext.readOnlyRootFilesystem: true` at the container level, paired with `emptyDir` volumes mounted at the handful of paths that need write access.

## Linux capabilities

Root's power on Linux isn't a single monolithic bit — it's split into roughly 40 distinct **capabilities**, each gating one category of privileged operation, checked independently by the kernel. A process can hold exactly the capabilities it needs without being "root" in the traditional sense. A few that come up constantly:

- `CAP_NET_BIND_SERVICE` — bind to ports below 1024 without being root.
- `CAP_CHOWN` — change file ownership.
- `CAP_DAC_OVERRIDE` — bypass file read/write/execute permission checks.
- `CAP_SYS_ADMIN` — a notorious catch-all covering many unrelated administrative operations (mounting filesystems and more); if you see a container asking for this, treat it as a red flag and find out exactly why.
- `CAP_SYS_PTRACE` — trace/inspect other processes' memory (debuggers, but also a powerful primitive for escaping containment).
- `CAP_NET_RAW` — open raw sockets (packet sniffing, crafting arbitrary packets).

Docker, when *not* run with `--privileged`, already grants a reduced default set (around 14 capabilities: `CHOWN`, `DAC_OVERRIDE`, `FOWNER`, `FSETID`, `KILL`, `SETGID`, `SETUID`, `SETPCAP`, `NET_BIND_SERVICE`, `NET_RAW`, `SYS_CHROOT`, `MKNOD`, `AUDIT_WRITE`, `SETFCAP`) — better than nothing, but still more than almost any application container needs.

The hardening pattern is to start from zero and add back only what's proven necessary:

```bash
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE myapp:latest
```

A huge fraction of real services — anything listening on a non-privileged port behind a reverse proxy or load balancer, which is most things in a modern architecture — needs **zero** added capabilities: `--cap-drop ALL` with no `--cap-add` at all is achievable and should be the starting assumption, not the exception.

Kubernetes equivalent:

```yaml
securityContext:
  capabilities:
    drop: ["ALL"]
    add: ["NET_BIND_SERVICE"]
```

## Seccomp profiles

Seccomp (secure computing mode) filters *which syscalls* a process is allowed to make, enforced at the kernel boundary — independent of and complementary to capabilities. Capabilities restrict what a privileged operation can do; seccomp restricts which syscalls can be invoked at all, privileged or not.

Docker applies a default seccomp profile to every container automatically (unless explicitly disabled), blocking around 44 syscalls considered dangerous or irrelevant inside a container — things like `mount`, `umount2`, `reboot`, `swapon`, `keyctl`, `add_key`, and restricting dangerous flag combinations on calls like `clone`. You can confirm it's active:

```bash
docker run --rm alpine grep Seccomp /proc/1/status
# Seccomp: 2   (2 = filtering active)
```

Disabling it entirely (`--security-opt seccomp=unconfined`) removes that layer completely — occasionally used for debugging with tracing tools that need syscalls the default profile blocks, never appropriate for production.

A custom, tighter profile can go further, allow-listing only the specific syscalls a given application actually calls and denying everything else by default:

```bash
docker run --security-opt seccomp=/path/to/custom-profile.json myapp:latest
```

Building a genuinely minimal profile means tracing the syscalls a working container makes under realistic load (via `strace`, or purpose-built tools like Inspektor Gadget's `traceloop`) and denying the rest — the gold-standard "default-deny with an explicit allowlist," though in practice most teams stick with Docker's shipped default rather than maintaining per-app profiles, because the maintenance overhead of a bespoke profile per service is real.

Kubernetes 1.19+: `securityContext.seccompProfile.type: RuntimeDefault` (the equivalent of Docker's shipped default) or `type: Localhost` pointing at a custom profile file distributed to every node.

## Points to Remember

- Containers share the host kernel — there is no hypervisor boundary. UID 0 inside a container is the kernel's real UID 0 unless a user namespace remaps it; capabilities and seccomp are what actually constrain it.
- `USER` in the Dockerfile must come before `CMD`/`ENTRYPOINT`; anything needing root privilege happens in earlier `RUN` steps.
- Kubernetes' `runAsNonRoot: true` is what makes non-root enforcement real — it refuses to start the container rather than silently falling back to root.
- `--read-only` + targeted `--tmpfs` mounts turns "identify every writable path" into a forcing function that usually reveals the app needs almost none.
- `--cap-drop ALL --cap-add <specific>` — start at zero, add back only what's proven necessary by actually running the workload, not by guessing.
- Seccomp and capabilities are complementary, not redundant: capabilities gate *what a privileged call can do*, seccomp gates *which syscalls can be made at all*.
- Docker's default seccomp profile is already on unless you explicitly disable it — the mistake to avoid is turning it off, not turning it on.

## Common Mistakes

- Shipping images that never set `USER`, relying entirely on `docker run --user` at deploy time — one forgotten flag anywhere in the deployment pipeline and the container silently runs as root again.
- Adding `CAP_SYS_ADMIN` (or leaving the full default capability set) because *one* feature needed *one* narrow permission, instead of finding the specific capability that feature actually requires.
- Treating `--privileged` as a quick fix for a permission error during development and shipping it to production — it disables almost every isolation mechanism covered here (all capabilities, seccomp, AppArmor/SELinux) at once.
- Setting `--read-only` and then discovering in production that the app crashes because a logging library, cache, or temp-file library writes somewhere you didn't declare a tmpfs mount for — test this under real load, not just a smoke test.
- Assuming Docker's default seccomp/capability restrictions are "enough" security on their own and skipping non-root and read-only hardening — they're one layer of several, not a complete answer by themselves.
- Disabling seccomp (`--security-opt seccomp=unconfined`) to make a debugging session easier and forgetting to remove the flag before deploying.
