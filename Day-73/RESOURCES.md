# Day 73 — Resources: Full CI/CD Pipeline Project

## Primary (assigned)

- **Your own notes** — the assigned "best resource" for today is deliberately your own accumulated Phase 2 notes. This capstone is meant to be an exercise in synthesis: go back through Days 61–72 and pull each concept (testing, SAST, container scanning, signing, GitOps) into one working pipeline rather than reading anything new.

## Deepen your understanding

- **GitHub Actions docs — "Using jobs in a workflow"** (docs.github.com/actions): the authoritative reference on `needs`, `if`, job outputs, and `permissions` scoping — everything the DAG design in file 1 depends on.
- **ArgoCD docs — "Sync Policies"** (argo-cd.readthedocs.io): explains `automated`, `prune`, `selfHeal`, and manual sync in full, plus `Application` CRD reference.
- **sigstore/cosign docs** (docs.sigstore.dev): keyless signing, OIDC identity binding, and how `cosign verify` validates against a certificate identity/issuer rather than a static public key.
- **OpenGitOps principles** (opengitops.dev): the CNCF-backed definition of what makes a deployment model "GitOps" — useful for articulating *why* pull-based reconciliation is the pattern, not just *how* to configure ArgoCD.

## Reference

- **`docker/build-push-action` README** (github.com/docker/build-push-action): the `outputs.digest` mechanism used to thread an immutable reference through scan/sign/push jobs.
- **Trivy Action README** (github.com/aquasecurity/trivy-action): every `severity`, `exit-code`, and SARIF-output option for gating a build on scan results.

## Practice

- **Awesome GitHub Actions** (github.com/sdras/awesome-actions): curated list of real-world workflow examples, including preview-environment and Slack-notification patterns to compare against your own implementation.
- **ArgoCD "Getting Started" interactive sandbox** (argoproj.github.io or Killercoda's ArgoCD scenarios): practice `syncPolicy` behavior and `selfHeal` without needing your own cluster first, if you want a dry run before Lab 3.
