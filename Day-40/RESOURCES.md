# Day 40 — Resources: K8s Observability & Debugging

## Primary (assigned)

- **Kubernetes troubleshooting guide** (kubernetes.io/docs/tasks/debug) — the assigned starting point. The official docs' "Debug Running Pods," "Debug Pods," and "Determine the Reason for Pod Failure" pages cover exactly the workflows in today's notes.

## Deepen your understanding

- **Kubernetes official docs — Debug Init Containers** and **Ephemeral Containers** — the authoritative reference for `kubectl debug` usage patterns, including `--target` and `--copy-to`.
- **"OOM Killed: A Guide to Kubernetes Memory Limits"** (Learnk8s or similar practitioner write-ups) — good visual walkthroughs of the cgroup OOM mechanism and requests-vs-limits distinction.
- **Kubernetes official docs — Kubernetes Events** — details on Event object structure, TTL configuration (`--event-ttl`), and how to query them effectively with `--field-selector`.

## Reference / lookup

- **`k9s` documentation** (k9scli.io) — full keybinding reference for the terminal UI used throughout today's lab.
- **`stern` GitHub repo** (github.com/stern/stern) — flag reference for multi-pod log tailing.
- **Exit code reference (Linux signals)** — `man 7 signal` on any Linux box lists every signal number; exit code minus 128 gives you the signal that killed the process.

## Practice

- **Complete today's lab (5 intentional breakages)** end to end on a real cluster (`kind`/`minikube`) — this is explicitly the assigned hands-on activity, and there's no substitute for having personally watched exit code 137 and a real `OOMKilled` reason appear in `describe` output.
- Pick any public "Kubernetes failure scenario" katas/challenges (several exist as open-source GitHub repos with deliberately broken manifests) and time yourself diagnosing each with only `kubectl` — building speed under a time constraint mirrors interview/on-call pressure.
