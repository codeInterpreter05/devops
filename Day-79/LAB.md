# Day 79 — Lab: Runtime Security

**Goal:** Deploy Falco against a real cluster, and — the assigned hands-on activity — write a custom rule to detect shell execution inside a container, then correlate that detection against the Kubernetes audit log and kernel-level preventive controls.

**Prerequisites:** A local Kubernetes cluster (`kind` or `minikube`; `kind` is used below since it makes audit-policy configuration straightforward via its config file), `helm`, and `kubectl`. Root/sudo access on the machine running the cluster (Falco's eBPF probe needs elevated node access).

---

### Lab 1 — Stand up a cluster with audit logging enabled

1. Create a `kind` config enabling the API server audit policy:
   ```yaml
   # kind-audit-config.yaml
   kind: Cluster
   apiVersion: kind.x-k8s.io/v1alpha4
   nodes:
     - role: control-plane
       kubeadmConfigPatches:
         - |
           kind: ClusterConfiguration
           apiServer:
             extraArgs:
               audit-policy-file: /etc/kubernetes/policies/audit-policy.yaml
               audit-log-path: /var/log/kubernetes/audit.log
             extraVolumes:
               - name: audit-policies
                 hostPath: /etc/kubernetes/policies
                 mountPath: /etc/kubernetes/policies
                 readOnly: true
               - name: audit-logs
                 hostPath: /var/log/kubernetes
                 mountPath: /var/log/kubernetes
                 readOnly: false
       extraMounts:
         - hostPath: ./audit-policy.yaml
           containerPath: /etc/kubernetes/policies/audit-policy.yaml
         - hostPath: ./audit-logs
           containerPath: /var/log/kubernetes
   ```
2. Write the `audit-policy.yaml` from today's third README (logging `pods/exec`/`pods/attach` and `secrets` at `RequestResponse`, everything else at `Metadata`).
3. `kind create cluster --config kind-audit-config.yaml`
4. Confirm audit events are being written: `docker exec -it <control-plane-container> tail -f /var/log/kubernetes/audit.log`

**Success criteria:** The audit log file is actively receiving JSON events as you run any `kubectl` command.

---

### Lab 2 — Deploy Falco and observe the default rule fire

1. Install Falco via Helm:
   ```bash
   helm repo add falcosecurity https://falcosecurity.github.io/charts
   helm repo update
   helm install falco falcosecurity/falco \
     --namespace falco --create-namespace \
     --set driver.kind=ebpf
   ```
2. Deploy a throwaway pod: `kubectl run test-pod --image=nginx`
3. Exec into it: `kubectl exec -it test-pod -- /bin/bash`
4. Watch Falco's logs in another terminal: `kubectl logs -n falco -l app.kubernetes.io/name=falco -f` — confirm the default "Terminal shell in container" rule fires, at `NOTICE` priority.
5. In parallel, `grep` the audit log for the matching `pods/exec` event and confirm the timestamp and `sourceIPs`/`user` fields line up with your Falco alert.

**Success criteria:** You have a Falco alert and a corresponding audit log entry, side by side, for the same `kubectl exec` action, with matching timestamps.

---

### Lab 3 — The core hands-on activity: write a custom Falco rule

1. Write a custom rules file targeting a specific scenario more precise than the default — e.g., flag any shell spawned in a namespace labeled `env=production` that isn't a known debug tool:
   ```yaml
   # custom-rules.yaml
   - macro: prod_namespace
     condition: k8s.ns.name = "production"

   - list: approved_debug_tools
     items: [kubectl-debug, dlv]

   - rule: Unexpected shell in production namespace
     desc: Interactive shell spawned in production, excluding known debug tooling.
     condition: >
       spawned_process and container and shell_procs
       and proc.tty != 0
       and prod_namespace
       and not proc.pname in (approved_debug_tools)
     output: >
       Unexpected shell in production (user=%user.name pod=%k8s.pod.name
       ns=%k8s.ns.name cmdline=%proc.cmdline)
     priority: CRITICAL
     tags: [container, shell, production]
   ```
2. Load it via Helm values (`--set-file customRules.rules-custom\\.yaml=custom-rules.yaml` or by mounting a ConfigMap, depending on your chart version) and upgrade the release.
3. Create a `production` namespace, deploy a pod there, and exec into it — confirm your custom rule fires at `CRITICAL`, distinct from the default `NOTICE`-level rule.
4. Exec into a pod in a *different* namespace and confirm your custom rule does **not** fire there (only the default rule does) — this proves your `prod_namespace` macro is scoping correctly.

**Success criteria:** Two visibly different alert priorities/messages depending on which namespace the shell was spawned in, proving your custom rule is correctly scoped and not just duplicating the default.

---

### Lab 4 — Apply a preventive seccomp profile and confirm it actually blocks something

1. Redeploy `test-pod` with an explicit `RuntimeDefault` seccomp profile:
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: hardened-pod
   spec:
     securityContext:
       seccompProfile:
         type: RuntimeDefault
     containers:
       - name: nginx
         image: nginx
         securityContext:
           allowPrivilegeEscalation: false
           capabilities: { drop: ["ALL"] }
   ```
2. Exec in and attempt a syscall `RuntimeDefault` blocks, e.g.: `unshare --mount echo test` — confirm it fails with a permission error, versus succeeding in a pod with no seccomp profile set (or `Unconfined`).
3. Note in writing which specific syscall was blocked and why that matters for limiting a compromised container's options.

**Success criteria:** A documented, reproduced difference in behavior for the same command between an unprofiled pod and a `RuntimeDefault`-profiled pod.

---

### Cleanup

```bash
kubectl delete pod test-pod hardened-pod --ignore-not-found
kubectl delete namespace production --ignore-not-found
helm uninstall falco -n falco
kind delete cluster
```

### Stretch challenge

Install Falcosidekick alongside Falco and wire your custom `CRITICAL` rule to post to a real (or test) Slack webhook, so the full path — kernel event → Falco rule match → Falcosidekick fan-out → chat alert — is demonstrated end to end, not just visible in `kubectl logs`.
