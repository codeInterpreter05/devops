# Day 120 — Resources: CKS Exam Prep

## Primary (assigned)

- **killer.sh CKS simulator** (killer.sh) — the assigned starting point and today's core hands-on activity. Two free sessions are bundled with every official CKS exam purchase, built by the same team delivering the real proctored exam environment, and deliberately calibrated to be at least as hard as the real thing.

## Deepen your understanding

- **Linux Foundation / CNCF — CKS Candidate Handbook and Exam Curriculum** (training.linuxfoundation.org) — the authoritative source for current task format, timing, passing score, curriculum domain weightings, the CKA prerequisite, and the exact allowed-documentation-domain list; always verify directly before your sitting since these details shift between exam versions.
- **kube-bench documentation and CIS Kubernetes Benchmark** (github.com/aquasecurity/kube-bench, cisecurity.org) — the full checklist your remediation practice should be grounded in, not just kube-bench's own summarized output.
- **Falco documentation — Rules** (falco.org/docs/rules) — the authoritative reference for rule/macro/list syntax, output field reference, and the default ruleset you should extend rather than edit.
- **Kubernetes docs — Seccomp and AppArmor** (kubernetes.io/docs/tutorials/security/seccomp, .../apparmor) — official tutorials for both, including the field-name differences between the current stable `appArmorProfile` field and the legacy annotation form.
- **Trivy and Trivy Operator documentation** (aquasecurity.github.io/trivy, aquasecurity.github.io/trivy-operator) — full CRD reference for `VulnerabilityReport`/`ConfigAuditReport`/etc., and the CLI's full scanner-type reference (image, fs, config, secret).

## Reference / lookup

- `kubectl explain <resource>` — fast in-terminal field reference during timed practice, no browser tab needed.
- **Kyverno Policy Library** (kyverno.io/policies) — a large set of pre-written, real-world admission policies (including seccomp/AppArmor enforcement and image verification) worth reading even if you don't deploy them directly, since CKS-style tasks often want a policy shaped very similarly to one of these.
- **sigstore / cosign documentation** (docs.sigstore.dev) — keyless signing model and Rekor transparency log, useful background for the "why digest, not tag" signing questions.

## Practice

- **kube-bench + a real `kubeadm` cluster** — free, repeatable CIS Benchmark drilling; a managed cluster hides too much of the control plane to be useful for the `master`/`etcd` targets.
- **Falco default ruleset on a `kind` cluster** — trigger real alerts (spawn a shell, read a sensitive file) rather than only reading the rules file, as in today's Lab 3.
- **killer.sh (your two bundled sessions)** — sequence deliberately: an early session right after finishing the curriculum as a cold baseline, and the second shortly before your actual exam date to confirm previously-found gaps are closed (same sequencing logic as the CKA sessions from Day 54).
