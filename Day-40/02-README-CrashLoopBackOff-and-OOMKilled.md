# Day 40 — K8s Observability & Debugging: CrashLoopBackOff & OOMKilled

**Phase:** 1 – Core DevOps | **Week:** W6 | **Domain:** Kubernetes | **Flag:** ⚡ Interview-critical

## Brief

`CrashLoopBackOff` is the single most common Kubernetes status a real operator will see, and it's also the exact wording of today's flagged interview question: *"A pod is stuck in CrashLoopBackOff. Walk me through your complete debugging process."* It isn't a root cause itself — it's a **symptom label** the kubelet applies when a container keeps exiting and restarting. A strong answer walks through a structured elimination process, not a guess. `OOMKilled` is one specific, very common root cause behind many CrashLoopBackOff pods and deserves its own deep dive.

## What `CrashLoopBackOff` actually means

It means: the container started, **exited** (for any reason — crash, or even a clean `exit 0` if it's not meant to be a long-running process), and the kubelet is now waiting an **exponentially increasing backoff delay** (10s, 20s, 40s... capped at 5 minutes) before trying again, per the pod's `restartPolicy` (default `Always` for pods managed by a Deployment). `CrashLoopBackOff` is the *status Kubernetes shows you while it's in that waiting period* — it is not itself a cause, which is precisely why "explain CrashLoopBackOff" as a standalone concept is a shallow answer; the interview question is really asking for the diagnostic *process*.

## The structured debugging process

1. **`kubectl get pods`** — confirm the status and note the **restart count**. A high, still-climbing restart count confirms an active loop versus a one-time crash.
2. **`kubectl describe pod <name>`** — check:
   - **`Last State`** and its **`Reason`** and **`Exit Code`** under container statuses (see exit code table below).
   - The **Events** section for anything Kubernetes itself observed (OOM kill, failed liveness probe, failed volume mount).
3. **`kubectl logs <name> --previous`** — this is where the actual application-level error usually lives (stack trace, missing environment variable, failed DB connection, uncaught exception at startup).
4. **Check resource requests/limits** (`describe` again, or `kubectl get pod <name> -o yaml | grep -A4 resources`) — is this container being OOMKilled because its memory limit is too low for what it actually needs?
5. **Check readiness/liveness probes** — an overly aggressive liveness probe (too short a timeout, hitting an endpoint before the app has finished initializing) can cause Kubernetes itself to kill an otherwise-healthy, slow-starting container repeatedly — this looks identical to an application crash from the outside but the "crash" is actually Kubernetes-initiated.
6. **Check dependencies** — is this container's startup blocked on something external (a database, another service, a ConfigMap/Secret that doesn't exist or is malformed) that's unavailable, causing it to crash at initialization every time?
7. **Reproduce locally if possible** — run the exact same image with the exact same env vars/config outside Kubernetes (`docker run`) to isolate whether the problem is the application itself or something Kubernetes-specific (networking, mounted volumes, resource limits).

## Reading exit codes — the fastest signal

```bash
kubectl describe pod myapp-x2j4k | grep -A5 "Last State"
```

| Exit code | Meaning |
|---|---|
| `0` | Clean exit — for a long-running app, this often means the process itself decided to stop (e.g., finished a one-shot task but is running as a Deployment expecting it to stay up) |
| `1` | Generic application error (uncaught exception, unhandled error) |
| `137` | `128 + 9` = killed by **SIGKILL** — almost always an **OOM kill** or an external force-kill (e.g., failed liveness probe past its threshold) |
| `139` | `128 + 11` = **SIGSEGV**, a segmentation fault — usually a lower-level bug (native code, bad memory access, sometimes a corrupted binary/incompatible architecture) |
| `143` | `128 + 15` = killed by **SIGTERM** — a graceful shutdown request, often the pod being terminated normally (scale-down, rolling update) rather than a crash |

Exit code **137** specifically is the fastest "is this an OOM kill" signal, before you even need to check `describe`'s `Reason` field (which will separately say `OOMKilled` when that's the specific cause detected by the kubelet).

## OOMKilled — root cause analysis

`OOMKilled` means the Linux kernel's **cgroup memory controller** killed the container's process because it exceeded its configured **memory limit** (`resources.limits.memory`). This is enforced at the **cgroup level by the kernel itself**, not a soft, negotiable Kubernetes-level decision — once a container's cgroup hits its memory limit, the kernel's OOM killer fires against that cgroup, full stop, regardless of node-level memory availability elsewhere.

```yaml
resources:
  requests:
    memory: "256Mi"
  limits:
    memory: "512Mi"
```

Diagnosing an OOM kill:

```bash
kubectl describe pod myapp-x2j4k    # look for "Reason: OOMKilled" and "Exit Code: 137"
kubectl top pod myapp-x2j4k --containers    # live memory usage (needs metrics-server)
```

Root causes behind an OOM kill fall into two buckets:
1. **The limit is genuinely too low** for the application's real working set — bump `limits.memory` after profiling actual usage under realistic load (`kubectl top`, or application-level memory metrics over time).
2. **A memory leak** — usage climbs monotonically over the container's lifetime rather than stabilizing, and no limit would be "enough" for long — this needs actual profiling of the application (heap dumps, language-specific memory profilers: `pprof` for Go, heap snapshots for Node.js/JVM) to find and fix the leak itself, not just a bigger limit as a band-aid.

Distinguishing the two: plot memory usage over the container's uptime (via `kubectl top` sampled repeatedly, or better, real metrics in Prometheus/Grafana). A sawtooth pattern that stabilizes under a ceiling and gets OOMKilled only under traffic spikes suggests "limit too low for peak load." A steady, unbroken upward climb that eventually always ends in an OOM kill, restart, then the same climb repeating, is the signature of a genuine leak.

### Requests vs. limits — why both matter here

`requests.memory` is what the scheduler uses to decide **which node** has room for this pod (bin-packing) — it does not cap actual usage. `limits.memory` is the actual hard ceiling enforced by the kernel cgroup, causing an OOM kill if exceeded. Setting `requests` far below real usage causes the scheduler to over-pack a node (many pods "fit" on paper but the node runs genuinely short on memory under real load, risking node-level memory pressure evictions across *unrelated* pods too — not just yours). Setting `limits` with no real profiling behind the number is effectively guessing at what will trigger an OOM kill.

## Points to Remember

- `CrashLoopBackOff` is a status describing the kubelet's exponential-backoff restart behavior, not a root cause — a good answer always follows with the actual diagnostic steps, not a definition.
- The debugging sequence: `get pods` (restart count) → `describe` (Last State/Reason/Exit Code + Events) → `logs --previous` (the actual error) → check resources/probes/dependencies → reproduce locally if needed.
- Exit code 137 = SIGKILL (almost always OOM or forced kill), 139 = SIGSEGV, 143 = SIGTERM (graceful). Memorizing these turns `describe` output into an instant diagnosis, before reading a single log line.
- OOMKilled is enforced by the kernel's cgroup memory controller against `limits.memory`, not a soft Kubernetes-level decision — it fires regardless of how much memory the node has free elsewhere.
- Distinguish "limit too low for real peak usage" (bump the limit, backed by profiling) from "genuine memory leak" (monotonically climbing usage that no limit fixes) by looking at the usage pattern over time, not just the fact that an OOM kill happened once.

## Common Mistakes

- Treating `CrashLoopBackOff` itself as the answer to "what's wrong," instead of using it as the starting symptom that triggers the actual investigation (`describe`, `logs --previous`, exit codes).
- Reflexively raising the memory limit every time an OOM kill happens without checking whether usage is stable (limit genuinely too tight) or continuously climbing (a real leak that a bigger limit only delays, not fixes).
- Confusing `requests` and `limits` — assuming a low `requests.memory` value protects against OOM kills, when it's `limits.memory` that the kernel enforces; `requests` only affects scheduling/bin-packing decisions.
- Not checking liveness probe configuration when a container appears to "crash" repeatedly — an app that's just slow to start, combined with too-short `initialDelaySeconds`/`failureThreshold`, gets killed by Kubernetes itself (often exit code 137, mistaken for an app bug or OOM) before it ever finishes initializing.
- Forgetting `--previous` on `kubectl logs` for a currently-crash-looping pod and concluding the logs are empty/unhelpful, when the meaningful error output is sitting in the terminated instance's logs, not the fresh restart's.
