# Day 70 — Artifact Management: Signing, Provenance, and the SLSA Framework

**Phase:** 2 – CI/CD & Security | **Week:** W11 | **Domain:** CI/CD

## Brief

Every practice from this week — scanning, signing, SBOMs, policy enforcement, registry hygiene — is really building toward one question: **can you prove, not just claim, how an artifact was built and that nothing tampered with it along the way?** SLSA (pronounced "salsa," Supply-chain Levels for Software Artifacts) is the framework that formalizes "prove" into concrete, auditable levels. "Explain SLSA, what Level 3 means and why it matters" is a real, current interview question specifically because SLSA gives you a shared vocabulary for supply-chain security maturity instead of vague claims like "we take security seriously."

## Provenance — the missing piece beyond signing and SBOMs

Recall the distinction from Day 67: a **signature** proves *who* produced an artifact and that it's unmodified since. An **SBOM** proves *what's inside* an artifact. **Provenance** is the third piece: a verifiable, structured record of **how** an artifact was built — which source commit, which build system, which build steps, what inputs, running as which identity. Provenance is what lets you answer "was this actually built by our CI pipeline from the commit we think it was, using the process we expect" instead of just "was this signed by *something* claiming to be our CI."

```bash
# Generating and attaching build provenance with Cosign (SLSA-compatible predicate type)
cosign attest --yes --predicate provenance.json --type slsaprovenance myimage@sha256:<digest>

# Verifying it later
cosign verify-attestation --type slsaprovenance myimage@sha256:<digest>
```

GitHub Actions has a built-in, first-party way to generate this without hand-rolling it: **`actions/attest-build-provenance`**, which produces a signed SLSA provenance attestation for a build automatically, tied to the specific workflow run, commit, and inputs that produced it.

```yaml
permissions:
  id-token: write
  attestations: write
  contents: read
steps:
  - uses: actions/checkout@v4
  - run: docker build -t myapp:${{ github.sha }} .
  - uses: actions/attest-build-provenance@v1
    with:
      subject-name: myregistry.io/myapp
      subject-digest: sha256:<digest>
```

## The SLSA levels — what each one actually requires

SLSA defines a progression of increasingly strong guarantees about the build process itself, not the artifact's code quality:

- **Level 0** — no guarantees. No provenance, ad-hoc/manual builds. Most legacy pipelines start here by default.
- **Level 1** — the build process is scripted/automated and produces provenance (a record of *how* it was built exists), but that provenance isn't independently verified or tamper-resistant. Better than nothing, but a compromised or careless CI job could still fake provenance.
- **Level 2** — builds run on a **hosted build platform** that generates provenance *itself*, not the build script — meaning the artifact's own build steps can't lie about how they were produced, because the trusted platform (not the untrusted build script) is the one asserting the facts. Provenance is also signed at this level.
- **Level 3** — the build platform provides **strong tamper resistance**: build environments are isolated per-build (no way for one build to influence another, e.g., no persistent, mutable, shared build cache/state that a prior malicious build step could poison), secrets used during the build aren't accessible to the build script's own logic in a way that could leak them into the provenance-signing process, and the provenance is **non-forgeable** even by someone with some access to the build system itself.

The practical jump most worth understanding for an interview: **Level 2 trusts the build platform to be honest** (it self-reports what happened); **Level 3 makes it structurally impossible for even a compromised build step to lie**, because the untrusted parts (your build script, your dependencies) are isolated from the trusted part (the platform's provenance generation) with real security boundaries, not just convention.

| Level | Guarantee | Example |
|---|---|---|
| 0 | None | Manual build on a laptop, pushed by hand |
| 1 | Provenance exists | A CI script that writes out a provenance file itself |
| 2 | Provenance is platform-generated and signed | GitHub Actions' `attest-build-provenance`, hosted CI generating its own attestation |
| 3 | Tamper-resistant, isolated, non-forgeable | Ephemeral, hardened, isolated build environments (e.g., GitHub Actions-hosted runners w/ hardened workflow, Google Cloud Build's SLSA3 builder) |

## Why this matters beyond a compliance checkbox

This isn't hypothetical: the **SolarWinds** attack (2020) succeeded by compromising the *build process* itself, injecting malicious code during compilation — not by exploiting the shipped source code. Signing and scanning the final artifact wouldn't have caught this, because the attacker's malicious step happened *inside* a build process that then produced a validly-signed, seemingly-clean artifact. This is exactly the failure mode SLSA Level 3 is designed to structurally prevent: an untrusted or compromised build step shouldn't be able to influence or lie about the trusted provenance record.

## Tying Day 67, 68, and 70 together

By this point in the week, a mature pipeline has: Trivy/Grype scanning (known vulnerabilities), Cosign signing by digest with keyless OIDC identity (integrity/authenticity), Syft-generated SBOM attestations (inventory), Kyverno/Gatekeeper admission enforcement (runtime gate), and now SLSA provenance attestations (build-process integrity) — each answering a different question, and together forming a real, defensible supply-chain security posture rather than any single control standing alone.

## Points to Remember

- Provenance answers "how was this built" (source, build system, steps, inputs) — distinct from signing ("who produced it, unmodified") and SBOMs ("what's inside it"). All three are needed together for real supply-chain assurance.
- SLSA levels measure trust in the *build process*, not code quality — Level 2 trusts a hosted platform to self-report honestly; Level 3 makes that self-report structurally non-forgeable via isolation and tamper resistance.
- The SolarWinds attack is the canonical real-world example of why build-process integrity (not just artifact scanning/signing) matters — the malicious code was injected *during* the build, producing an artifact that would have passed a scan and a naive signature check.
- `actions/attest-build-provenance` is GitHub's first-party, ready-to-use way to generate SLSA-compatible provenance without hand-building the attestation yourself.
- A mature supply-chain security posture layers scanning + signing + SBOM + admission enforcement + provenance together — each control covers a gap the others don't.

## Common Mistakes

- Conflating "we sign our images" with "we have SLSA Level 3" — signing alone says nothing about whether the build environment itself was isolated/tamper-resistant; it's necessary but not sufficient for higher SLSA levels.
- Assuming SLSA Level 1 (self-reported provenance from the build script) is meaningfully strong — a compromised build script can simply write whatever provenance content it wants at Level 1, since nothing outside the untrusted script verifies or generates it.
- Treating SLSA adoption as all-or-nothing — realistically, most orgs incrementally move from Level 0/1 toward Level 2/3 for their highest-value pipelines first, not everything at once; presenting it as binary in an interview undersells practical rollout thinking.
- Not isolating build environments between runs (shared, long-lived, mutable build agents/caches) while still claiming strong provenance guarantees — a shared cache poisoned by one build can silently influence a later, supposedly-independent build's output, which is exactly what Level 3 isolation is meant to prevent.
- Forgetting that provenance needs to be *verified* by consumers (via `cosign verify-attestation` or an admission-time policy check), not just generated — generating an unused attestation provides no actual protection if nothing downstream ever checks it before deploying.
