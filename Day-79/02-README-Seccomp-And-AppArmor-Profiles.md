# Day 79 — Runtime Security II: Seccomp & AppArmor Profiles

**Phase:** 2 – CI/CD & Security | **Week:** W13 | **Domain:** DevSecOps | **Flag:** —

## Brief

Falco (previous file) is **detective** — it tells you something bad happened. Seccomp and AppArmor are **preventive** — kernel-enforced restrictions that shrink what a compromised container can even attempt in the first place, regardless of whether anything is watching. This is defense in depth in its purest form: if an attacker gets code execution inside a container, a tight seccomp/AppArmor profile is what stands between "ran an arbitrary command" and "successfully escalated privileges, mounted the host filesystem, or pivoted to the node." Interviewers ask about these specifically because they separate candidates who've only read about container security from those who've actually configured a `securityContext` and understood why each field is there.

## Seccomp — restricting which syscalls a process can make

**Seccomp (secure computing mode)** filters the set of system calls a process is allowed to invoke. Every container "does things" exclusively by asking the kernel via syscalls (`open`, `write`, `execve`, `mount`, `ptrace`, `keyctl`, …) — a process that can't make a dangerous syscall in the first place can't use it in an exploit, no matter how compromised the userspace code is.

Two operating modes:
- **Strict mode**: only `read`, `write`, `exit`, and `sigreturn` are allowed — extremely restrictive, rarely usable for real application containers.
- **Filter mode**: a BPF (this is the *original* BPF that gave eBPF its name) program evaluates each syscall attempt against an allow/deny list and returns an action per syscall — allow, deny with an error (`EPERM`), or kill the process.

**In Kubernetes**, seccomp is configured per pod/container via `securityContext.seccompProfile`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hardened-app
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: myapp:1.0
      securityContext:
        allowPrivilegeEscalation: false
        capabilities:
          drop: ["ALL"]
        readOnlyRootFilesystem: true
        runAsNonRoot: true
```

- **`RuntimeDefault`** uses the container runtime's built-in default profile (containerd/CRI-O ship one based on Docker's original default), which blocks roughly 40-plus syscalls considered dangerous for containerized workloads — things like `mount`, `reboot`, `keyctl`, `add_key`, `ptrace` (in most defaults), `unshare` in certain combinations, and kernel-module-related calls. This alone meaningfully shrinks the attack surface with essentially zero application compatibility cost for the vast majority of normal workloads.
- **`Localhost`** points to a custom JSON profile file staged on every node at `/var/lib/kubelet/seccomp/profiles/<name>.json`, letting you write an even tighter allow-list tailored to exactly the syscalls your specific application actually needs — the gold standard, but it requires generating and maintaining a profile per application (tools like `oci-seccomp-bpf-hook` or running the app under `strace -f -c` in a test environment can help enumerate the real syscall set to allow-list).

A minimal custom profile shape:

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    {
      "names": ["read", "write", "open", "close", "exit_group", "epoll_wait", "futex"],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

`defaultAction: SCMP_ACT_ERRNO` means anything not explicitly allow-listed returns an error instead of executing — the safe default-deny posture.

## AppArmor — path-based mandatory access control

Where seccomp restricts *which syscalls* a process can call, **AppArmor** restricts *what files, paths, and capabilities* a program can touch, regardless of the file's own Unix permission bits. It's a **Linux Security Module (LSM)** — the same kernel framework, incidentally, that eBPF's LSM hooks (used by Tetragon) also plug into — implementing **Mandatory Access Control (MAC)**: even if a file is world-readable at the Unix-permissions level, an AppArmor profile can still deny a specific process from reading it, because MAC policy is enforced independently of and on top of discretionary Unix permissions.

Profiles are loaded into the kernel by name and reference paths directly, e.g., "this profile allows read access to `/etc/myapp/*` but denies write access to anything under `/etc` and denies network raw sockets entirely."

**In Kubernetes**, historically this was wired up via an annotation:

```yaml
metadata:
  annotations:
    container.apparmor.security.beta.kubernetes.io/app: localhost/my-custom-profile
```

As of Kubernetes 1.30 (GA), this moved to a proper structured field, which you should prefer in new manifests:

```yaml
spec:
  containers:
    - name: app
      securityContext:
        appArmorProfile:
          type: Localhost
          localhostProfile: my-custom-profile
```

Like seccomp's `Localhost` type, this requires the named profile to already be loaded on the node (via `apparmor_parser` or your node's bootstrap/AMI configuration) — Kubernetes itself doesn't ship or load AppArmor profiles for you, it only references ones already present.

**SELinux**, common on RHEL-family distros, is the other major LSM-based MAC system and solves a similar problem to AppArmor with a different model — label-based rather than path-based (every process and file gets a security label, and policy defines which labels can interact with which), generally considered more powerful but noticeably more complex to author policy for. Know that it exists and is the RHEL-world's default answer to the same class of problem AppArmor solves on Debian/Ubuntu-family systems — this distinction itself is a reasonable interview probe ("what's the difference between AppArmor and SELinux").

## Putting it together: a hardened pod spec

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hardened-app
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    fsGroup: 10001
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: myapp:1.0
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
        appArmorProfile:
          type: RuntimeDefault
```

This single spec stacks four independent layers: non-root UID (limits what host-level damage a compromise can do), seccomp (limits syscalls), dropped Linux capabilities (limits privileged operations even as root inside the container), and a read-only root filesystem (limits an attacker's ability to write a persistent payload into the container's own filesystem) — each one closing a different class of escape or persistence technique, which is exactly why "defense in depth" isn't just a buzzword here: no single control is sufficient alone, but the combination meaningfully raises the bar.

## Points to Remember

- Seccomp filters **syscalls** (what a process can ask the kernel to do); AppArmor filters **paths/capabilities** (what files and privileged operations a process can touch) — different enforcement axes, and both are useful simultaneously.
- `RuntimeDefault` (for both seccomp and AppArmor) is a strong, low-effort default that blocks the clearly-dangerous syscalls/operations most workloads never legitimately need — apply it everywhere as a baseline before considering custom profiles.
- `Localhost`-type profiles require the actual profile file to exist on the node already — Kubernetes references and enforces a profile, it does not generate or ship one for you.
- SELinux (RHEL-family) and AppArmor (Debian/Ubuntu-family) solve the same MAC problem with different models — label-based versus path-based — know which one your distro/base image actually uses.
- These are preventive controls (reduce blast radius before/regardless of detection) — pair them with Falco (detective) and audit logs (forensic record) for genuine defense in depth, not a substitute for either.

## Common Mistakes

- Leaving `seccompProfile` unset entirely (defaults vary by cluster/runtime configuration and Kubernetes version — don't assume `RuntimeDefault` is applied automatically everywhere) instead of setting it explicitly at the pod or namespace level.
- Writing an extremely permissive custom seccomp/AppArmor profile "to make the app work" during initial rollout, and never tightening it afterward once the real required syscall/path set is actually known — the temporary permissive profile quietly becomes permanent.
- Assuming a `Localhost` profile will just work across the cluster without confirming the profile file is actually staged on *every* node (including new nodes added by autoscaling) — a profile missing on one node causes pods to fail to start there with a confusing error, not a security failure but an availability one.
- Treating "runs as non-root" alone as sufficient hardening — a non-root user with a permissive seccomp profile and no capability drops can still do meaningful damage; these controls are complementary, not substitutes for one another.
- Confusing AppArmor/seccomp (preventive, kernel-enforced, always-on) with Falco (detective, alert-based) in an interview answer — a strong answer distinguishes "this stops the action" from "this tells you the action happened," and explains why you want both.
