# Day 73 — Full CI/CD Pipeline Project: Pipeline Architecture

**Phase:** 2 – CI/CD & Security | **Week:** W12 | **Domain:** CI/CD | **Flag:** 📌 Capstone

## Brief

Everything you've learned across Phase 2 — testing, SAST, container builds, vulnerability scanning, signing, GitOps — has been taught as isolated topics. In the real world, none of it exists in isolation: a production pipeline is a single connected chain where each stage's output is the next stage's input, and a failure anywhere upstream must block everything downstream. Today is the capstone: wire test → SAST → build → scan → sign → push into one coherent GitHub Actions pipeline for your Phase 1 infrastructure project. This is also the single most common "walk me through your pipeline" interview question — the ability to narrate a full pipeline end-to-end, including *why* each gate exists and what happens when one fails, is what separates someone who has used CI/CD from someone who has only configured individual steps.

This day is split into three files:

1. **This file** — designing the pipeline as a directed graph of stages, job dependencies, and fail-fast gating.
2. **[02-README-GitOps-Deployment.md](02-README-GitOps-Deployment.md)** — ArgoCD auto-sync to staging vs. manual promotion to production.
3. **[03-README-Notifications-Preview-Envs.md](03-README-Notifications-Preview-Envs.md)** — Slack failure notifications and PR preview environments.

## Designing the pipeline as a dependency graph

A production pipeline isn't a flat script — it's a **DAG (directed acyclic graph)** of jobs. In GitHub Actions, `needs:` expresses these edges. The stage order for this capstone is:

```
test → sast → build → scan → sign → push → (deploy handled by ArgoCD, see file 2)
```

Each arrow is a **hard gate**: if `test` fails, `sast` never runs; if `scan` finds a critical CVE, `sign`/`push` never happen. This is deliberate — the entire point of gating is that a vulnerable or broken artifact should be *structurally incapable* of reaching a registry, not merely "flagged."

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read
  id-token: write      # needed for OIDC-based cloud auth and keyless signing
  security-events: write # needed to upload SARIF to GitHub code scanning

env:
  IMAGE_NAME: ghcr.io/${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - run: npm test -- --coverage
      - uses: actions/upload-artifact@v4
        with: { name: coverage, path: coverage/ }

  sast:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Semgrep
        uses: returntocorp/semgrep-action@v1
        with:
          config: p/owasp-top-ten
      - name: Run CodeQL
        uses: github/codeql-action/init@v3
        with: { languages: javascript }
      - uses: github/codeql-action/analyze@v3

  build:
    needs: sast
    runs-on: ubuntu-latest
    outputs:
      digest: ${{ steps.build.outputs.digest }}
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - id: build
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: ${{ env.IMAGE_NAME }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  scan:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Scan image with Trivy
        uses: aquasecurity/trivy-action@0.24.0
        with:
          image-ref: ${{ env.IMAGE_NAME }}@${{ needs.build.outputs.digest }}
          severity: CRITICAL,HIGH
          exit-code: '1'          # fail the job — this IS the gate
          format: sarif
          output: trivy-results.sarif
      - uses: github/codeql-action/upload-sarif@v3
        if: always()
        with: { sarif_file: trivy-results.sarif }

  sign:
    needs: scan
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      packages: write
    steps:
      - uses: sigstore/cosign-installer@v3
      - name: Keyless sign with cosign
        run: |
          cosign sign --yes ${{ env.IMAGE_NAME }}@${{ needs.build.outputs.digest }}

  push:
    needs: sign
    runs-on: ubuntu-latest
    steps:
      - run: echo "Image already pushed in build step; tag as release-ready"
      - run: |
          docker buildx imagetools create \
            --tag ${{ env.IMAGE_NAME }}:latest \
            ${{ env.IMAGE_NAME }}@${{ needs.build.outputs.digest }}
```

Notice the artifact digest (`needs.build.outputs.digest`), not the mutable tag, flows through `scan`, `sign`, and `push`. This is not incidental style — **tags are mutable pointers, digests are immutable content addresses.** If you scan-and-sign by tag, a race (or a compromised registry) could swap the underlying image between your scan and your sign step. Pinning to digest closes that gap — this is the same principle as pinning third-party GitHub Actions to a commit SHA instead of a tag (`uses: actions/checkout@<sha>` vs `@v4`), which supply-chain-conscious pipelines do for the same reason.

## Why job separation (not one giant job) matters

You could cram everything into a single job with sequential `run:` steps. Splitting into separate jobs buys you:

- **Parallelism where it's safe.** `sast` (static analysis of source) and later stages that only need the *artifact* don't have to share a runner — GitHub Actions can schedule independent jobs on separate runners concurrently once their `needs` are satisfied, cutting wall-clock time.
- **Clear failure attribution.** The GitHub UI shows which named job failed — "scan failed" is a much better signal (to you and to Slack, see file 3) than "step 14 of a 40-step job failed."
- **Fine-grained permissions.** The `sign` job needs `id-token: write` for keyless OIDC signing; `test` doesn't. Scoping `permissions:` per-job (rather than repo-wide) is least-privilege applied to CI itself.
- **Independent caching and artifact reuse.** Build outputs, coverage reports, and SARIF files are passed via `actions/upload-artifact` / job outputs rather than re-computed.

## Fail-fast vs. fail-safe stages

Not every stage should hard-fail the pipeline. A mature pipeline distinguishes:

- **Blocking gates** — test failures, SAST criticals, scan criticals/highs, missing signature. These use `exit-code: 1` or default job failure behavior, and downstream jobs simply never run because `needs` isn't satisfied.
- **Informational stages** — license scanning, SBOM generation, code-coverage-delta reporting. These often run with `continue-on-error: true` or upload results without failing the build, because blocking merges on *every* finding (including low-severity noise) trains engineers to bypass the pipeline rather than fix things.

Getting this balance wrong in either direction is a real failure mode: too permissive and vulnerable code ships; too strict (blocking on every low/medium finding) and engineers start adding `if: false` to steps or requesting broad SAST-tool suppressions just to unblock merges — which defeats the entire point.

## Points to Remember

- The stage order `test → sast → build → scan → sign → push` is a hard dependency chain (`needs:` in GitHub Actions) — a failure at any stage must structurally prevent later stages from running, not just log a warning.
- Pass the immutable image **digest** between build/scan/sign/push jobs, not the mutable tag — this closes a TOCTOU gap where the image could change between scanning and signing.
- Split stages into separate jobs (not one long job) for parallelism, clearer failure attribution in the UI/notifications, and per-job least-privilege `permissions:` scoping.
- Distinguish blocking gates (fail the build) from informational stages (report but don't block) — over-blocking trains engineers to route around the pipeline.
- `permissions:` should be minimal per job — only the `sign` job needs `id-token: write`, not the whole workflow.

## Common Mistakes

- Building one giant job with everything in `run:` steps — a single flaky network call anywhere kills the whole thing with a useless "step 23 failed" message, and there's no parallelism.
- Scanning/signing by mutable tag (`:latest`, `:main`) instead of digest — the image referenced can change between steps, defeating the purpose of the gate.
- Setting `permissions: write-all` at the workflow level "to make errors go away" instead of diagnosing which specific permission each job actually needs.
- Making every single check (including cosmetic linting) a hard-blocking gate — this leads to `# noqa` / suppression comments proliferating everywhere as engineers route around friction instead of fixing root causes.
- Forgetting `needs:` on a job, so it runs in parallel with (rather than after) a gate it should depend on — a classic copy-paste bug when adding a new stage to an existing workflow.
