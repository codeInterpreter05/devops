# Day 80 — Lab: Phase 2 Project — Full Pipeline

**Goal:** Build, deploy, load test, and publish a complete DevSecOps pipeline — GitHub → CI → ECR → Argo CD → EKS, every security gate wired in and demonstrable — with a documented README as the final capstone artifact for Phase 2. This is a multi-day-effort project condensed into one lab; timebox aggressively and prefer "smaller but fully working end-to-end" over "ambitious but half-finished."

**Prerequisites:** A GitHub account, an AWS account (or substitute `kind` + a local registry if you want to avoid cloud cost — noted per lab where it matters), `docker`, `kubectl`, `helm`, `argocd` CLI, `k6`, `cosign`, `syft`, `trivy` installed locally for iterating before pushing to CI.

---

### Lab 1 — Scaffold the repo structure

1. Create a new public GitHub repo with this layout:
   ```
   app/                  # sample app (anything simple — a health-check API is enough)
   .github/workflows/    # ci.yml
   manifests/            # base + overlays (or a Helm chart) for Argo CD to sync
   policies/             # Kyverno ClusterPolicy YAMLs
   loadtest/             # k6 script
   README.md
   ```
2. Write a minimal app with at least one real endpoint (`/api/health`) and a `Dockerfile`.
3. Commit and push the initial scaffold.

**Success criteria:** A public repo exists with the directory structure above and a container image that builds locally (`docker build .` succeeds).

---

### Lab 2 — Build the CI pipeline with every security gate wired in

1. Implement `ci.yml` from today's first README: lint/test → SAST (Semgrep) → dependency scan → build → Trivy scan (gate) → SBOM (Syft) → sign (cosign) → push to ECR, using `needs:` to chain them.
2. Create the ECR repo (`aws ecr create-repository --repository-name app`) and wire the CI job's credentials via GitHub Actions secrets (or, better, OIDC federation to an IAM role — no long-lived AWS keys in GitHub secrets).
3. **Prove the gate actually gates:** intentionally add a dependency with a known critical CVE (or introduce an obviously insecure code pattern Semgrep's `p/owasp-top-ten` ruleset catches), push a PR, and confirm the pipeline fails at that specific job — not later, not silently.
4. Fix the issue, confirm the pipeline goes green end-to-end and an image lands in ECR.

**Success criteria:** A screenshot or log showing the pipeline failing on an intentionally-introduced vulnerability, and a second run showing it passing and pushing a signed, scanned image to ECR.

---

### Lab 3 — Wire up Argo CD and admission policy

1. Install Argo CD on your EKS (or `kind`) cluster: `kubectl create namespace argocd && kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml`
2. Create the `Application` manifest from today's first README pointing at your `manifests/` path, with `syncPolicy.automated.selfHeal: true`.
3. Manually edit a deployed resource with `kubectl edit` and confirm Argo CD reverts it automatically (proving `selfHeal` — a genuinely convincing demo of GitOps as the actual source of truth).
4. Install Kyverno and apply the `disallow-privileged-and-root` policy from today's first README. Attempt to deploy a pod with `privileged: true` and confirm it's rejected at admission time, with the policy's message shown in the error.

**Success criteria:** A manual cluster edit gets auto-reverted by Argo CD, and a deliberately-insecure pod spec is rejected by Kyverno with a clear policy violation message.

---

### Lab 4 — Load test the deployed app

1. Adapt the k6 script from today's second README to hit your actual deployed endpoint.
2. Run it while watching `kubectl get hpa -w` and `kubectl top pods` in parallel.
3. Record: p95/p99 latency, error rate, and how many seconds elapsed between the spike stage starting and the HPA adding a new replica.
4. If anything OOMKilled or crash-looped during the spike, note it honestly — this is a legitimate, useful finding for the README, not a failure of the lab.

**Success criteria:** A written record of real p95/p99 latency numbers and HPA scale-out timing from an actual run against your deployed app.

---

### Lab 5 — The core hands-on activity: write the README and publish

1. Write the full README using the 8-section structure from today's second README (overview, Mermaid architecture diagram, tech stack table, security controls table, how to reproduce, load test results, one real tradeoff/problem, future improvements).
2. Make sure the Mermaid diagram renders correctly on GitHub (preview it in the repo, not just locally).
3. Push everything, confirm the repo is public, and pin it on your GitHub profile.

**Success criteria:** A published, public repo with a complete README that a stranger could read in 90 seconds and understand what was built, why, and what you learned — reusing the exact audit standard from Day 60.

---

### Cleanup

```bash
# Tear down cloud resources to avoid ongoing cost
kubectl delete application my-app -n argocd
eksctl delete cluster --name <your-cluster>     # or: kind delete cluster
aws ecr delete-repository --repository-name app --force
```

### Stretch challenge

Open a real PR against your own finished repo that intentionally reintroduces the vulnerability from Lab 2, and record a short screen capture (or a sequence of screenshots in the README) showing the full gate-to-rejection flow end to end — this is a far more convincing portfolio artifact for an interviewer to actually watch than a static architecture description.
