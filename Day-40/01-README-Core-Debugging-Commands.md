# Day 40 — K8s Observability & Debugging: Core Debugging Commands

**Phase:** 1 – Core DevOps | **Week:** W6 | **Domain:** Kubernetes | **Flag:** ⚡ Interview-critical

## Brief

Anyone can `kubectl apply` a manifest that works. What separates a junior from a senior Kubernetes operator is what happens next: a pod stuck in a weird state, and no obvious error message. "Walk me through your debugging process for a broken pod" is asked in almost every DevOps/SRE interview because it's the fastest way to tell whether someone has actually operated a cluster under pressure or only followed tutorials where everything works on the first try. Today builds the toolkit — the specific commands, in the specific order, that a competent operator reaches for — before the next file goes deep on the two most common concrete failure modes (CrashLoopBackOff, OOMKilled).

This day is split into three files:

1. **This file** — the core kubectl debugging toolkit: `describe`, `logs`, `exec`, `port-forward`, events, and ephemeral debug containers.
2. **[02-README-CrashLoopBackOff-and-OOMKilled.md](02-README-CrashLoopBackOff-and-OOMKilled.md)** — deep root-cause analysis of the two most common pod failure states.
3. **[03-README-ImagePullBackOff-and-Common-Failures.md](03-README-ImagePullBackOff-and-Common-Failures.md)** — `ImagePullBackOff` and other frequent failure patterns.

## `kubectl describe` — always step one

```bash
kubectl describe pod myapp-7d9f8c6b5-x2j4k -n production
```

`describe` is the single highest-value first command for almost any Kubernetes problem — it aggregates the pod spec, current status, container statuses (including `state`, `lastState`, restart count), resource requests/limits, volume mounts, and — critically — the **Events** section at the bottom, which is a chronological log of everything the scheduler/kubelet did (or tried and failed) with this object: scheduling decisions, image pulls, liveness/readiness probe failures, OOM kills. Most real Kubernetes debugging sessions are solved or at least correctly diagnosed from `describe` output alone, before you even need logs.

## `kubectl logs` — what the container itself said

```bash
kubectl logs myapp-7d9f8c6b5-x2j4k                       # current container's stdout/stderr
kubectl logs myapp-7d9f8c6b5-x2j4k -c sidecar             # specific container in a multi-container pod
kubectl logs myapp-7d9f8c6b5-x2j4k --previous              # logs from the PREVIOUS (crashed) instance
kubectl logs myapp-7d9f8c6b5-x2j4k -f                       # follow/stream, like tail -f
kubectl logs myapp-7d9f8c6b5-x2j4k --since=10m              # only the last 10 minutes
kubectl logs -l app=myapp --all-containers --prefix          # across every pod matching a label selector
```

The single most-forgotten flag here is **`--previous`**. When a container crashes and restarts, `kubectl logs` (no flag) shows the *current, freshly-restarted* container's logs — which, if it crashed again quickly, may show almost nothing useful. `--previous` retrieves the log output from the **last terminated instance**, which is almost always where the actual crash reason (a stack trace, an uncaught exception, an assertion failure) lives.

For following logs across many pods at once (a deployment with several replicas, or several related pods), plain `kubectl logs -f` only follows one pod. **`stern`** solves this natively:

```bash
stern myapp -n production                 # tail logs from every pod matching "myapp", multiplexed, color-coded per pod
stern myapp --since 5m --tail 50
```

## `kubectl exec` — get an interactive shell inside the container

```bash
kubectl exec -it myapp-7d9f8c6b5-x2j4k -- /bin/sh
kubectl exec myapp-7d9f8c6b5-x2j4k -- env
kubectl exec myapp-7d9f8c6b5-x2j4k -- cat /etc/resolv.conf
```

`exec` runs a command *inside an already-running container*, using whatever binaries exist in that container's image. This is where minimal/distroless images (common for security-hardened production images with no shell, no package manager, no coreutils) become a real debugging obstacle — `kubectl exec -it ... -- /bin/sh` simply fails with "executable file not found" if the image has no shell at all. This exact limitation is why **ephemeral debug containers** exist (below).

## `kubectl port-forward` — reach a pod/service without exposing it externally

```bash
kubectl port-forward pod/myapp-7d9f8c6b5-x2j4k 8080:80
kubectl port-forward svc/myapp 8080:80
kubectl port-forward deployment/myapp 8080:80
```

Forwards a local port to a port inside the cluster, tunneled through the API server — useful for hitting an internal service/database directly from your laptop to test connectivity or inspect an internal-only admin/metrics endpoint, without setting up an Ingress, a NodePort, or a LoadBalancer just for a one-off debugging session. It only lasts as long as the foreground process runs (Ctrl+C ends it) and is explicitly a debugging tool, not a production traffic path.

## Kubernetes Events and their TTL

Events (visible standalone via `kubectl get events`, or embedded in `describe` output) are short-lived, cluster-level records of things that happened to an object — scheduling, pulling images, probe failures, OOM kills, volume mount issues.

```bash
kubectl get events -n production --sort-by='.lastTimestamp'
kubectl get events -n production --field-selector involvedObject.name=myapp-7d9f8c6b5-x2j4k
kubectl get events -n production -w      # watch new events as they happen
```

Critically, **events have a default TTL of about 1 hour** (`--event-ttl` on the kube-apiserver, default `1h0m0s`) — after which they're garbage collected from etcd and gone forever, even though the pod/object itself may still exist. This is why "the pod crashed 3 hours ago and I want to see why" often comes up empty on `kubectl get events` even though it would have clearly shown the reason within that first hour. For anything you need to investigate after the fact, you need a **persistent** events/logging pipeline (shipped to something like Loki, Elasticsearch, or CloudWatch Logs) — relying on live cluster events alone for post-incident analysis doesn't work once enough time has passed.

## Ephemeral debug containers — debugging minimal/distroless images

```bash
kubectl debug -it myapp-7d9f8c6b5-x2j4k --image=busybox --target=myapp
kubectl debug node/ip-10-0-1-23 -it --image=busybox    # debug a NODE, not just a pod
kubectl debug myapp-7d9f8c6b5-x2j4k -it --image=busybox --copy-to=myapp-debug --container=myapp
```

An **ephemeral container** is injected into an *already-running* pod without restarting it, sharing the same network namespace (and, with `--target`, the same process namespace) as the existing containers — so you get a shell with actual debugging tools (`curl`, `netstat`, `strace`) even though the app's own container image has none of that. `--copy-to` instead creates a *new* pod that's a copy of the original with the debug container added — useful when you don't want to touch the live, possibly-serving-traffic pod at all. This is the modern (stable since Kubernetes 1.25) replacement for the old hack of building a special "debug" variant of your production image just so it would have a shell.

## Points to Remember

- `kubectl describe` is almost always the correct first command — its Events section usually tells you what actually happened before you need to read a single log line.
- `kubectl logs --previous` is the single most useful, most forgotten flag — it retrieves logs from the container instance that just crashed, not the fresh restart that replaced it.
- `kubectl exec` requires a shell/binary to already exist in the target image; minimal/distroless production images often have none, which is exactly the gap `kubectl debug` (ephemeral containers) fills.
- Kubernetes Events expire (default ~1 hour TTL) — for post-incident analysis beyond that window, you need logs/events shipped to persistent external storage, not just `kubectl get events`.
- `kubectl port-forward` is a debugging tunnel through the API server, not a production traffic path — it dies when the foreground command is interrupted.

## Common Mistakes

- Running `kubectl logs` without `--previous` on a crash-looping pod and concluding "there's nothing in the logs" — the current instance may have crashed again almost immediately with minimal output, while `--previous` holds the actual crash detail.
- Trying `kubectl exec -it ... -- /bin/sh` on a distroless/minimal image and treating the "executable file not found" error as a permissions or connectivity problem, instead of recognizing the image simply has no shell — the fix is `kubectl debug`, not more exec flag-fiddling.
- Assuming `kubectl get events` will always show what happened, without accounting for the ~1 hour default TTL — investigating an incident hours later and finding no events doesn't mean nothing happened; it means the record already expired.
- Forgetting `-n <namespace>` and debugging against the wrong namespace's identically-named pod (very common in multi-tenant or multi-environment clusters), wasting time investigating a healthy pod while the actually-broken one goes unexamined.
- Using `kubectl port-forward` as a long-term access method (e.g., leaving it running for a whole team to share) instead of a proper Ingress/Service — it's fragile (single point of failure, tied to one person's terminal session) and not meant for that purpose.
