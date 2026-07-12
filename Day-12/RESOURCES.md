# Day 12 — Resources: Container Security

## Primary (assigned)

- **OWASP Docker Security Cheat Sheet** (cheatsheetseries.owasp.org) — free, the assigned starting point for today. Covers the same ground as today's three README files (non-root, capabilities, read-only filesystems, image provenance) from OWASP's security-first angle, and is the single most commonly cited reference when this topic comes up in interviews.

## Deepen your understanding

- **Trivy documentation** (aquasecurity.github.io/trivy) — official docs covering every scan target (image, filesystem, repo, Kubernetes cluster, IaC) in more depth than today's cheatsheet, including CI integration examples for every major CI provider.
- **Sigstore documentation** (docs.sigstore.dev) — explains Fulcio, Rekor, and the keyless-signing model from the project that builds Cosign; worth reading directly rather than only through Cosign's own docs, since it explains *why* the architecture works the way it does.
- **Cosign documentation** (docs.sigstore.dev/cosign/overview) — command reference for signing, verifying, and attesting, plus registry-compatibility notes.
- **Docker's own container security documentation** (docs.docker.com/engine/security) — covers capabilities, seccomp, AppArmor, and user namespaces from the engine's own reference docs; useful for confirming exact default behavior (e.g., the precise default capability list) straight from the source.

## Reference / lookup

- **CIS Docker Benchmark** (CIS Center for Internet Security) — the industry-standard checklist auditors and security teams actually use to score a Docker host/image configuration; skimming it once gives you a mental checklist beyond what any single tool checks for you.
- **`man 7 capabilities`** — run this on any Linux box; it's the kernel's own authoritative list of every capability and exactly what it grants, useful when you hit a capability name you don't recognize in a scan or policy.

## Practice

- **Trivy GitHub Action** (`aquasecurity/trivy-action`) — drop this into a real GitHub Actions workflow to practice the CI-gating pattern from today's stretch challenge with `--exit-code 1` for real, not just locally.
- **killercoda.com container-security scenarios** — free, browser-based, guided scenarios for exactly this domain (capabilities, seccomp, image scanning) if you want more guided repetition beyond today's lab before moving on.
