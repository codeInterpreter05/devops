# Day 120 — CKS Exam Prep: Runtime Security — Audit Logs, Falco, Seccomp & AppArmor

**Phase:** 4 – Advanced/Specialization | **Week:** W21 | **Domain:** Advanced Security | **Flag:** 📌 Milestone

## Brief

Every control covered so far this phase — RBAC, NetworkPolicy, mTLS, admission control — is *preventive*: it tries to stop something bad from happening. Runtime security starts from a different assumption: **something already got in.** A container is compromised, a malicious process spawned, a supply-chain attack slipped a backdoored dependency past your scanners. The question runtime security answers is "can you detect it, and can you limit what it's able to do once it's there?" This is genuinely the most operationally relevant domain of the four covered today for real incident response — hardening prevents, runtime security detects and contains — and it's also the domain most CKS candidates under-practice, because it requires hands-on time with tools (Falco, seccomp, AppArmor) that don't come up nearly as often in day-to-day cluster administration as RBAC or NetworkPolicy do.

## Kubernetes audit logging

The API server can log every request it receives — audit logging is how you answer "who did what, to what, and when" after the fact, which is the foundation both attackers and forensic investigators care about.

**Stages** (a single request generates events at up to four stages): `RequestReceived`, `ResponseStarted`, `ResponseComplete`, `Panic`.

**Levels** (increasing verbosity/storage cost): `None` (drop it), `Metadata` (who/what/when, no body), `Request` (+ request body), `RequestResponse` (+ response body — most verbose, most expensive).

An audit **Policy** resource defines rules matching requests by user, group, resource, namespace, or verb, mapping each match to a level:

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["secrets"]
    verbs: ["get", "list"]
  - level: Metadata
    omitStages: ["RequestReceived"]
  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
```

**Rule order matters — first match wins**, and there's no implicit catch-all: if nothing matches a request, it isn't logged at all. Always end a real policy with a broad fallback rule if you want anything beyond your explicit high-value rules captured.

Wiring this up means adding flags to the **kube-apiserver static pod manifest**: `--audit-policy-file`, `--audit-log-path`, `--audit-log-maxage`, `--audit-log-maxbackup`, `--audit-log-maxsize` for a file-based backend, or `--audit-webhook-config-file` to ship events to an external SIEM instead. **The CKS-relevant trap:** adding these flags to `/etc/kubernetes/manifests/kube-apiserver.yaml` without also adding matching `volumeMounts`/`volumes` entries for the policy file and log directory causes the apiserver static pod to crash-loop on its kubelet-triggered restart — silently, with no `kubectl` command available to diagnose it, since the API server itself is the thing that's down.

Querying the resulting JSON-lines log is a `jq` exercise:

```bash
cat /var/log/kubernetes/audit/audit.log | \
  jq 'select(.verb=="delete" and .objectRef.resource=="secrets")'
```

## Falco: runtime threat detection

**Falco** (CNCF) taps directly into the kernel's syscall stream — via a kernel module, an eBPF probe (the modern default, no out-of-tree module compilation needed per node), or a ptrace-based fallback — attaches Kubernetes/container metadata to each syscall event, and evaluates that stream against a rules engine in real time.

A rule has four parts: `rule` (name), `desc`, `condition` (a boolean expression over fields like `proc.name`, `fd.name`, `k8s.pod.name`, `container.id`, `evt.type`), and `output` (the alert message template), plus a `priority`.

```yaml
- rule: Terminal shell in container
  desc: A shell was spawned by a program in a container with an attached terminal.
  condition: >
    spawned_process and container
    and shell_procs and proc.tty != 0
    and container.id != host
  output: >
    Shell spawned in container (user=%user.name container_id=%container.id
    container_name=%container.name shell=%proc.name parent=%proc.pname
    cmdline=%proc.cmdline)
  priority: WARNING
```

`shell_procs` here is a built-in **list** (a reusable named set of values, e.g. shell binary names) — rules compose from **macros** (reusable condition fragments) and lists so you're not repeating the same boilerplate condition across dozens of rules. **Never edit the shipped default ruleset directly** (`falco_rules.yaml`) — it gets overwritten on upgrade. Add or override rules in a separate local file (conventionally `falco_rules.local.yaml`), and use `append: true` when extending an existing macro/list rather than redefining it, or you'll silently replace the built-in definition instead of adding to it.

In Kubernetes, Falco runs as a **DaemonSet** because it needs host-level visibility (`/proc`, the kernel/eBPF probe, the container runtime socket) — one instance per node observing that node's syscalls. This is also why the resource footprint is worth planning for: it's watching every syscall on every node, all the time, not just Kubernetes API traffic.

## Seccomp: syscall filtering

**Seccomp** (secure computing mode) is a Linux kernel feature that filters *which syscalls* a process is allowed to make, enforced via a BPF filter the kernel loads per-process. It reduces attack surface directly: even if an attacker achieves code execution inside a container, if the syscall their exploit chain needs — `mount`, `ptrace`, `unshare`, and similar syscalls commonly abused for container-breakout attempts — is blocked, the exploit simply fails at that step.

Kubernetes exposes it via `securityContext.seccompProfile`:

- **`RuntimeDefault`** — the container runtime's built-in default profile, moderately restrictive, a safe baseline for most workloads, and the profile CKS wants you to move workloads *toward* from the historical default.
- **`Localhost`** — a custom profile JSON file that must already exist on every node the pod could schedule to, at a kubelet-configured path (commonly `/var/lib/kubelet/seccomp/profiles/<name>.json`).
- **`Unconfined`** — no filtering at all. This was the historical implicit default, and it's exactly what CKS-style hardening tasks want you to eliminate.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secured-pod
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      seccompProfile:
        type: Localhost
        localhostProfile: profiles/audit.json
```

Writing a custom profile from scratch without observing real behavior is how you break the app — some syscalls are only exercised on rarely-hit code paths (error handling, signal handling) and blocking one your app legitimately needs elsewhere causes an intermittent, hard-to-reproduce failure. The practical workflow is to run the workload under an audit/permissive profile (or trace it with Falco/`strace`) first, record the syscalls it actually makes, and build an allowlist from *that* — not from guessing.

**Cluster-wide enforcement** (rather than per-pod hope) is done via Pod Security Admission at the `restricted` level, or a Kyverno/OPA policy asserting `seccompProfile.type != Unconfined` on every pod — CKS scenarios test both the per-pod mechanism and the cluster-wide policy that guarantees it's actually applied everywhere.

## AppArmor: mandatory access control

**AppArmor** is a Linux Security Module (LSM) providing **MAC (mandatory access control)**: profiles restrict what files, network operations, and Linux capabilities a *named binary/path* can use — enforced regardless of the process's own UID or in-container permissions. This is what makes it "mandatory": even root *inside* the container is still bound by a profile enforced from outside the container's own control, unlike ordinary discretionary file permissions the process's own UID could otherwise satisfy.

**Seccomp vs. AppArmor — the distinction CKS wants you to articulate cleanly:** seccomp restricts *which syscalls* can be called at all (the kernel API surface itself); AppArmor restricts *what those syscalls are allowed to act on* (specific paths, specific network operations, specific capabilities). They're complementary layers, not redundant — a process might be allowed to call `open()` (seccomp permits the syscall) but AppArmor can still deny it access to a specific sensitive path.

The profile must be **loaded on the node itself** first (`apparmor_parser -q -r /path/to/profile`, typically shipped via node bootstrap tooling or a DaemonSet), then referenced from the pod spec via `securityContext.appArmorProfile` (the stable field since Kubernetes 1.30 — older clusters/exam versions may instead show the deprecated `container.apparmor.security.beta.kubernetes.io/<container>` annotation; know both forms, since exam material can lag the newest field):

```yaml
securityContext:
  appArmorProfile:
    type: Localhost
    localhostProfile: k8s-nginx-restricted
```

**The gotcha that matters in both the exam and production:** if the named profile is loaded on only one node in a multi-node cluster and the pod isn't pinned to that node (via `nodeSelector`/`nodeAffinity`), the pod's success or failure to start becomes **node-dependent and intermittent** — it works fine when scheduled to the node with the profile, and fails with a confusing error when scheduled anywhere else, which is a much harder failure mode to diagnose than an outright, consistent rejection.

## Points to Remember

- Runtime security assumes a breach already happened somewhere and asks whether you can detect and contain it — a fundamentally different posture from the preventive controls (RBAC, NetworkPolicy, admission control) covered elsewhere.
- Audit logging: Stages (RequestReceived/ResponseStarted/ResponseComplete/Panic) and Levels (None/Metadata/Request/RequestResponse) are independent axes; policy rules match first-hit-wins with no implicit catch-all, so unmatched requests are silently not logged.
- Falco taps the kernel syscall stream (via eBPF/kernel module/ptrace), evaluates rules built from conditions, macros, and lists — always extend via a local rules file, never edit the shipped default ruleset.
- Seccomp filters *which syscalls* a process may call; AppArmor (MAC) restricts *what those syscalls can act on* (paths/network/capabilities) — complementary, not redundant, layers.
- Both seccomp `Localhost` and AppArmor `Localhost` profiles require the profile file to exist on the scheduled node *before* the pod starts — an unpinned pod can land on a node without the profile and fail unpredictably.

## Common Mistakes

- Adding audit-log flags to the kube-apiserver static pod manifest without the matching `volumeMounts`/`volumes` for the policy file and log directory, crash-looping the API server with no `kubectl`-visible error.
- Writing an audit Policy with only high-value specific rules and no broad fallback rule, then being surprised that unrelated requests generate zero audit events at all — there's no default "log everything not otherwise matched."
- Editing Falco's shipped default rules file directly instead of adding overrides/extensions in a local rules file, then losing all customizations silently on the next Falco upgrade.
- Writing a custom seccomp profile from imagination rather than from observed syscall behavior, breaking the application on a rarely-exercised code path (typically error handling) that wasn't captured during testing.
- Loading an AppArmor or Localhost seccomp profile onto only one node in a multi-node cluster and not pinning the pod's scheduling to it, producing intermittent, node-dependent pod startup failures.
