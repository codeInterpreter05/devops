# Day 79 — Runtime Security I: Falco & eBPF-Based Threat Detection

**Phase:** 2 – CI/CD & Security | **Week:** W13 | **Domain:** DevSecOps | **Flag:** —

## Brief

Every scanner you've used so far in this phase (SAST, dependency scanning, container image scanning) is a **pre-deploy** control — it answers "is there a *known* problem in this artifact before it ships." None of that helps once a container is already running and something unexpected happens: a zero-day, a compromised dependency that only misbehaves at runtime, an attacker who got in through a legitimate but stolen credential, or simply a misconfigured pod doing something it shouldn't. Runtime security is the detective layer that watches what's *actually happening* inside running workloads and alerts (or blocks) on suspicious behavior in real time. This is exactly the domain of today's assigned interview question — "how would you detect if someone exec'd into a production container and ran commands" — and Falco, built on eBPF, is the industry-standard open-source answer.

This day is split into three files:

1. **This file** — how Falco and the underlying eBPF technology actually work, with a real custom rule.
2. **[02-README-Seccomp-And-AppArmor-Profiles.md](02-README-Seccomp-And-AppArmor-Profiles.md)** — kernel-enforced *preventive* controls (seccomp, AppArmor) that reduce what an attacker can do even before detection kicks in.
3. **[03-README-K8s-Audit-Logs-And-Incident-Response.md](03-README-K8s-Audit-Logs-And-Incident-Response.md)** — the authoritative API-server record of who did what, and how to actually respond when Falco fires.

## eBPF: how you get kernel-level visibility without writing a kernel module

**eBPF (extended Berkeley Packet Filter)** is a kernel technology that lets you load small, sandboxed programs into the Linux kernel at runtime, attached to specific hook points — **kprobes** (dynamic hooks on almost any kernel function), **tracepoints** (static, stable hook points the kernel maintainers guarantee), and **LSM hooks** (Linux Security Module hooks, the same framework SELinux/AppArmor use, which is what lets eBPF-based tools actually *block* an action, not just observe it after the fact).

Why this matters and why it replaced older approaches:
- Before eBPF, deep kernel-level observability meant either writing an actual **kernel module** (unsafe — a bug can crash the whole machine, and it has to be recompiled per kernel version) or using heavyweight tracing like `ptrace`/`auditd`/`strace`, which is comparatively very slow because every observed syscall requires a context switch back into userspace to be evaluated.
- eBPF programs run **inside the kernel**, are checked by a **verifier** before being loaded (rejecting anything with unbounded loops, out-of-bounds memory access, or other unsafe patterns — this is what makes "safely run arbitrary code in the kernel" a real, shipped capability instead of a security nightmare), and are JIT-compiled to near-native speed. The result: high-fidelity, low-overhead visibility that's practical to run on every production node, all the time — which older approaches generally were not.

## Falco's architecture

Falco (a CNCF graduated project) has three conceptual layers:

1. **Instrumentation** — either a kernel module, or (the modern default) an **eBPF probe**, collects raw system call and kernel event data from every process on the node.
2. **The Falco engine (libscap/libsinsp)** in userspace enriches that raw event stream with context — which container it came from, which Kubernetes pod/namespace/labels, which process tree — and evaluates it against a **rules file**.
3. **Rules** are declarative YAML conditions written over fields like `proc.name`, `container.id`, `evt.type`, `fd.name` — when a rule's condition matches an event, Falco emits an alert at a configured priority (e.g., `NOTICE`, `WARNING`, `CRITICAL`).

Falco ships a broad default ruleset (`falco_rules.yaml`) covering common attack patterns out of the box: a shell spawned inside a container, a write below `/etc`, an outbound connection to an unexpected port, a process reading cloud credential files, and — directly relevant to today's interview question — shell execution.

## A real rule: detecting shell execution inside a container

This is (close to) Falco's actual shipped default rule, and it's exactly the mechanism behind "how would you detect someone exec'd into a container":

```yaml
- rule: Terminal shell in container
  desc: >
    A shell was spawned by a program in a container with an attached
    terminal — this is the classic signature of an interactive
    `kubectl exec`/`docker exec` session.
  condition: >
    spawned_process
    and container
    and shell_procs
    and proc.tty != 0
    and container.id != host
  output: >
    A shell was spawned in a container with an attached terminal
    (user=%user.name container_id=%container.id container_name=%container.name
    shell=%proc.name parent=%proc.pname cmdline=%proc.cmdline terminal=%proc.tty
    image=%container.image.repository)
  priority: NOTICE
  tags: [container, shell, mitre_execution]
```

Reading this rule mechanically: `spawned_process` and `container` are conditions checking the event type and that it happened inside a container namespace; `shell_procs` is a Falco **list/macro** matching common shell binary names (`bash`, `sh`, `zsh`, …); `proc.tty != 0` is the key signal — a process with an attached pseudo-terminal is almost always an *interactive* session, which is what `kubectl exec -it` creates, as opposed to a script running a shell non-interactively as part of normal application behavior.

A genuinely custom variant — narrowing this to only fire for your production namespace, and excluding a legitimate debug tool your team uses — demonstrates you understand rule tuning, not just copy-pasting the default:

```yaml
- macro: prod_namespace
  condition: k8s.ns.name = "production"

- rule: Unexpected shell in production namespace
  desc: Interactive shell spawned in a production-labeled namespace, excluding known debug tooling.
  condition: >
    spawned_process and container and shell_procs
    and proc.tty != 0
    and prod_namespace
    and not proc.pname in (approved_debug_tools)
  output: >
    Unexpected shell in production (user=%user.name pod=%k8s.pod.name
    ns=%k8s.ns.name cmdline=%proc.cmdline)
  priority: CRITICAL
  tags: [container, shell, production, mitre_execution]
```

## Deployment and downstream alerting

Falco typically runs as a **DaemonSet** — one Falco pod per node, since it needs to observe kernel events for every container on that specific node. It needs elevated privileges (either a privileged security context or specific capabilities plus access to the eBPF probe/kernel headers) — a legitimate reason a security-conscious cluster grants an exception to an otherwise-strict Pod Security Admission policy, scoped narrowly to the Falco DaemonSet's service account.

Raw Falco output alone (stdout/syslog) doesn't page anyone. **Falcosidekick** is the standard companion that fans alerts out to real destinations — Slack, PagerDuty, a SIEM, or an object store for long-term retention — based on rule priority, so a `NOTICE` might just get logged while a `CRITICAL` pages on-call immediately.

## eBPF beyond detection: Tetragon

**Tetragon** (from Cilium/Isovalent) is a newer, eBPF-native alternative that goes a step further than Falco's detect-and-alert model: because it hooks in via LSM hooks, it can **enforce in-kernel**, blocking or killing the offending syscall *before it completes*, not just alerting after the fact. The tradeoff is real: enforcement that's too aggressive can break legitimate workloads, so Tetragon policies are usually rolled out in "alert-only" mode first, observed, and only promoted to enforcing mode once tuned — the same maturity curve any admission-control or firewall policy goes through.

## Points to Remember

- Runtime security is a **detective** (and, with Tetragon-style enforcement, occasionally preventive) control that complements pre-deploy scanning — it catches what static analysis structurally cannot: zero-days, compromised credentials, and unexpected live behavior.
- eBPF lets verified, sandboxed programs run inside the kernel at hook points (kprobes, tracepoints, LSM hooks) with near-native performance — this is what makes always-on, node-wide security observability practically feasible.
- Falco's core loop: eBPF/kernel-module probe collects syscalls → userspace engine enriches with container/K8s context → rules evaluate the enriched event stream → alerts fire at a configured priority.
- The concrete answer to "how do you detect an exec into a container": a Falco rule matching `proc.tty != 0` inside a container (an interactive shell was spawned) — pair this with the Kubernetes audit log (file 3) for the authoritative record of *who* issued the `exec` request.
- Falco alone doesn't route anywhere useful without Falcosidekick (or equivalent) fanning alerts out to Slack/PagerDuty/a SIEM based on priority.

## Common Mistakes

- Deploying Falco with only the default ruleset and never tuning it for your own environment — this produces alert fatigue from noisy but benign matches (e.g., a legitimate debug sidecar that spawns a shell as part of normal operation) until people start ignoring Falco alerts entirely.
- Granting Falco's DaemonSet broad, undocumented privileged access without narrowing the exception specifically to what the eBPF probe needs, which becomes its own audit finding ("why does this one pod bypass our Pod Security policy, and is that reviewed").
- Treating Falco as sufficient on its own without correlating its alerts against the Kubernetes audit log — Falco tells you *that* a shell appeared; the audit log tells you *which authenticated user* issued the `kubectl exec` request that caused it. You need both for a real incident timeline.
- Rolling out Tetragon (or any eBPF enforcement tool) directly into blocking mode without an observe-only tuning period first — legitimate workloads can get killed by an overly broad policy, which is how security tooling earns a reputation for "breaking production" and gets disabled by frustrated teams.
- Confusing eBPF-based tools with older syscall-tracing approaches (`strace`/`auditd`) in an interview answer — being able to explain *why* eBPF is lower-overhead (in-kernel execution, verifier-checked safety, no per-event context switch) is what distinguishes a candidate who's actually read about the mechanism from one who's only used the tool.
