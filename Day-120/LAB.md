# Day 120 — Lab: CKS Exam Prep

**Goal:** Drill every tool named in today's row (kube-bench, Falco, Trivy Operator, AppArmor) hands-on against real output — not just read about them — then run the assigned core hands-on activity: a full timed CKS mock exam on killer.sh, with weak areas logged for follow-up study.

**Prerequisites:** A `kubeadm`-provisioned cluster for the kube-bench/audit-log labs (kube-bench's master/etcd checks need a real control plane you administer — a managed cluster or plain `kind` won't expose the same files/flags). `kind` is fine for the Falco/seccomp/AppArmor/Trivy Operator labs. Helm 3, Docker running. A killer.sh session (bundled free with an active CKS exam registration, two sessions).

---

### Lab 1 — kube-bench: run it, read it, fix it

1. Install and run kube-bench against your kubeadm control-plane node:
   ```bash
   curl -L https://github.com/aquasecurity/kube-bench/releases/latest/download/kube-bench_linux_amd64.tar.gz -o kube-bench.tar.gz
   tar -xvf kube-bench.tar.gz
   sudo ./kube-bench run --targets master,etcd
   ```
2. Pick 3 FAIL results from the output. For each, without looking anything up first, write down what you think the fix is — then check the kube-bench-provided remediation text and compare.
3. Apply one real fix: locate the etcd data directory (`grep data-dir /etc/kubernetes/manifests/etcd.yaml`) and confirm/fix its permissions to `700`:
   ```bash
   sudo chmod 700 /var/lib/etcd
   ```
4. Re-run kube-bench and confirm that specific check now shows PASS.

**Success criteria:** You've turned at least one real kube-bench FAIL into a PASS, and you can explain the security reasoning behind it (not just "the tool told me to"), not just pasted the remediation.

---

### Lab 2 — Enable audit logging and query it with `jq`

1. On the control-plane node, create an audit policy at `/etc/kubernetes/audit-policy.yaml` using the example from `02-README-Runtime-Security-Falco-Seccomp-AppArmor-Audit.md`.
2. Edit `/etc/kubernetes/manifests/kube-apiserver.yaml`: add `--audit-policy-file=/etc/kubernetes/audit-policy.yaml` and `--audit-log-path=/var/log/kubernetes/audit/audit.log`, **and** add matching `volumeMounts`/`volumes` entries for both the policy file and the log directory (this is the step people skip — do it deliberately here so you feel the failure mode once safely).
3. Wait for the kubelet to restart the static pod (`watch kubectl get pod -n kube-system kube-apiserver-<node>`), then generate some audit-worthy activity: `kubectl create secret generic test-secret --from-literal=key=value`, then `kubectl get secret test-secret`.
4. Query the log:
   ```bash
   sudo cat /var/log/kubernetes/audit/audit.log | \
     jq 'select(.objectRef.resource=="secrets")'
   ```
5. **Deliberately break it once:** comment out just the `volumeMounts` entry for the log directory (leave the flag in place), save, and watch what happens to the apiserver static pod. Then fix it back.

**Success criteria:** You found your own `get secret` request in the audit log via `jq`, and you personally observed (and recovered from) the crash-loop caused by a flag with no matching volume mount.

---

### Lab 3 — Falco: install it, write a custom rule, trigger it

1. Install Falco via Helm on your `kind` cluster:
   ```bash
   helm repo add falcosecurity https://falcosecurity.github.io/charts && helm repo update
   helm install falco falcosecurity/falco --namespace falco --create-namespace \
     --set driver.kind=modern_ebpf
   ```
2. Tail Falco's alerts: `kubectl logs -n falco -l app.kubernetes.io/name=falco -f`.
3. Trigger a default rule for real: `kubectl run shell-test --image=busybox -it --rm -- sh` — confirm Falco fires a "Terminal shell in container"-style alert the moment you get a shell.
4. Add the custom rule from `02-README-Runtime-Security-Falco-Seccomp-AppArmor-Audit.md` (or a variant of your own) to a `falco_rules.local.yaml` provided via the Helm chart's `customRules` value, targeting a specific sensitive file read (e.g., `/etc/shadow`) instead of shell spawning. Redeploy, and trigger it: `kubectl exec shell-test -- cat /etc/shadow`.

**Success criteria:** You wrote and triggered one custom rule that isn't just a copy of a default rule with the name changed, and you can explain what `macro`/`list` reuse is for.

---

### Lab 4 — Seccomp and AppArmor, enforced and observed

1. Deploy a pod with `seccompProfile.type: RuntimeDefault`, then attempt a syscall the default profile blocks (e.g., `unshare -r` inside the container) and confirm it's denied where it would succeed unconfined.
2. Write a minimal AppArmor profile denying write access to `/tmp` for a container's process, load it on the node (`apparmor_parser -q -r <profile>`), and reference it via `securityContext.appArmorProfile` (or the legacy annotation if your cluster is pre-1.30). Confirm a write to `/tmp` inside the container fails while other operations still work.
3. Deliberately schedule the same pod on a *second* node where you have not loaded the AppArmor profile (use `nodeSelector` to force it), and observe the failure mode — this is the intermittent, node-dependent failure called out in the README.

**Success criteria:** You've seen a seccomp-blocked syscall and an AppArmor-denied file write fail for two genuinely different reasons, and reproduced the "profile not loaded on this node" failure on purpose.

---

### Lab 5 — Trivy Operator: continuous in-cluster scanning

1. Install:
   ```bash
   helm repo add aqua https://aquasecurity.github.io/helm-charts/ && helm repo update
   helm install trivy-operator aqua/trivy-operator \
     --namespace trivy-system --create-namespace --set="trivy.ignoreUnfixed=true"
   ```
2. Deploy a deliberately old, vulnerable image (e.g., `nginx:1.16`) and wait for the operator to generate a report.
3. Query it:
   ```bash
   kubectl get vulnerabilityreports -A
   kubectl get vulnerabilityreports -A -o json | \
     jq '.items[] | select(.report.summary.criticalCount > 0) | .metadata.name'
   ```
4. Replace the image with a current, patched tag and confirm a new report is generated with a lower (ideally zero) critical count.

**Success criteria:** You can point at a `VulnerabilityReport` object and explain, from the actual report contents, exactly which CVE triggered the highest severity finding.

---

### Lab 6 — The core hands-on activity: full timed CKS mock exam on killer.sh

1. Start (or resume) a killer.sh CKS session. Set a strict **2-hour timer**, matching real exam conditions — no pausing.
2. Attempt the full simulator scenario set. If a task isn't clicking within a few minutes, flag it and move on rather than stalling — same time-management discipline as CKA (Day 54).
3. Score yourself strictly against the provided solutions — partial credit only where the object would actually pass a real grader.
4. For every miss, write one sentence identifying the *actual* misconception (e.g., "I didn't realize `PeerAuthentication` precedence meant a namespace-level policy could override my mesh-wide STRICT setting"), not just "I forgot a flag."
5. Map each miss back to one of the six CKS curriculum domains from `01-README-CKS-Exam-Format-And-Domains.md` — this tells you exactly where to spend your remaining study time before the real exam.

**Success criteria:** A written list of misses with root causes and domain mapping — the domain mapping is what turns a raw score into an actual study plan.

---

### Cleanup

```bash
helm uninstall falco -n falco --ignore-not-found
helm uninstall trivy-operator -n trivy-system --ignore-not-found
kubectl delete pod shell-test --ignore-not-found
kind delete cluster --name cks-lab 2>/dev/null || true
# on the kubeadm node, if you want to fully revert audit logging:
# edit /etc/kubernetes/manifests/kube-apiserver.yaml back to remove the --audit-* flags and volume entries
```

### Stretch challenge

Chain three controls into one scenario the way a real CKS task would: deploy a pod with `Unconfined` seccomp and no AppArmor profile, write a Kyverno policy that rejects any pod without `seccompProfile.type: RuntimeDefault` at admission time (`validationFailureAction: Enforce`), redeploy the pod and confirm it's rejected before it ever schedules, then add a Falco rule that would have detected the specific dangerous syscall that pod's `Unconfined` profile would otherwise have permitted — proving the same risk is covered by prevention (Kyverno) and detection (Falco) independently.
