# Day 67 — Container & Registry Security: SBOM Generation with Syft

**Phase:** 2 – CI/CD & Security | **Week:** W11 | **Domain:** DevSecOps | **Flag:** ⚡ Interview-critical

## Brief

A Software Bill of Materials (SBOM) is a complete, machine-readable inventory of everything that went into building an artifact — every OS package, every language dependency, down to exact versions. It answers a question that becomes urgent the moment a new CVE drops: "**do we run this anywhere, and if so, where?**" Without an SBOM, answering that means manually re-scanning every image you've ever built. With one, it's a query. SBOMs went from a nice-to-have to effectively mandatory after Log4Shell (2021) and US Executive Order 14028, which requires SBOMs for software sold to the federal government — this is now a real compliance requirement teams get asked about in interviews, not just a security nicety.

## What Syft actually produces

Syft (also from Anchore, the same team behind Grype) inspects a container image, filesystem, or archive and outputs a structured list of every package it finds, in one of two industry-standard formats:

- **SPDX** — the more compliance/legal-oriented standard (originated for license auditing), widely required in regulated/government contexts.
- **CycloneDX** — originated in the AppSec/OWASP world, slightly more focused on vulnerability-management tooling and increasingly the default in security-first pipelines.

```bash
# Generate an SBOM for a built image, SPDX JSON format
syft myapp:latest -o spdx-json > sbom.spdx.json

# CycloneDX format instead
syft myapp:latest -o cyclonedx-json > sbom.cdx.json

# Human-readable table for a quick look
syft myapp:latest -o table

# Scan a directory/filesystem instead of an image (e.g. a checked-out repo before it's even built)
syft dir:. -o table

# Filter to a specific package type only
syft myapp:latest -o table --select-catalogers "python,npm"
```

A single image typically yields hundreds to low-thousands of catalogued packages once you count transitive dependencies (a "hello world" Node.js app dragging in its full `node_modules` tree, plus the OS packages in the base image, easily crosses a thousand entries) — this is exactly why doing it by hand doesn't scale and tooling is required.

## Attaching the SBOM to the image (attestations)

An SBOM sitting in a CI artifact bucket is useful, but the stronger pattern is attaching it *to the image itself* as a signed attestation, so anyone who pulls the image can also pull (and verify) its SBOM directly from the registry:

```bash
# Generate the SBOM and attach it to the image as a Cosign attestation, signed
syft myapp@sha256:<digest> -o cyclonedx-json > sbom.cdx.json
cosign attest --yes --predicate sbom.cdx.json --type cyclonedx myapp@sha256:<digest>

# Anyone can later verify the attestation and pull the SBOM back out
cosign verify-attestation --type cyclonedx myapp@sha256:<digest>
```

This ties directly into Day 67's earlier two topics: the SBOM is generated in the same CI pipeline that scans (Trivy/Grype) and signs (Cosign) the image — one pipeline stage producing scan results, a signature, and an attestable SBOM as three linked artifacts for the same digest.

### GitHub Actions example — full chain in one job

```yaml
- name: Generate SBOM
  uses: anchore/sbom-action@v0
  with:
    image: myregistry.io/myapp:${{ github.sha }}
    format: cyclonedx-json
    output-file: sbom.cdx.json

- name: Scan SBOM for vulnerabilities (reuse SBOM instead of re-scanning the image)
  run: grype sbom:./sbom.cdx.json --fail-on critical

- name: Attest SBOM to image
  run: cosign attest --yes --predicate sbom.cdx.json --type cyclonedx myregistry.io/myapp@${{ steps.build.outputs.digest }}
```

Notice `grype sbom:./sbom.cdx.json` instead of `grype myregistry.io/myapp:...` — feeding Grype an already-generated SBOM avoids a second, redundant image pull-and-unpack, which matters once images get large.

## Why SBOMs matter beyond "compliance checkbox"

- **Incident response speed**: when a CVE like Log4Shell drops, `grep`-ing across a database of stored SBOMs ("which images contain `log4j-core` version `< 2.17`?") turns a multi-day fire drill into a five-minute query — *if* you already have SBOMs generated and stored for every image you've shipped.
- **License auditing**: SBOMs also catalogue license metadata (GPL, MIT, Apache-2.0, etc.) — legal/compliance teams use this to catch a copyleft-licensed dependency that shouldn't be bundled into proprietary software.
- **Supply chain provenance**: combined with signing (previous file) and SLSA levels (Day 70), an SBOM is one of the building blocks of being able to prove — not just claim — what's actually inside a shipped artifact.

## Points to Remember

- SBOM = complete package inventory of an artifact (OS + language dependencies, with exact versions) — it answers "what's in here," not "is it vulnerable" (that's a separate scan step, though scanners can consume an SBOM as input).
- Two dominant formats: **SPDX** (compliance/legal-oriented) and **CycloneDX** (security-tooling-oriented) — know both names even if your org standardizes on one.
- Generate SBOMs at build time and store/attach them — regenerating an SBOM for an old image later only tells you what's in it *now* if the image hasn't changed, but doesn't help you retroactively search history unless you archived it originally.
- Attach the SBOM to the image itself via a signed Cosign attestation (`cosign attest`) rather than only stashing it as a CI artifact — this makes the SBOM pullable and verifiable by anyone downstream, not just accessible to whoever has CI access.
- A scanner (Grype/Trivy) can consume a pre-generated SBOM directly (`grype sbom:./sbom.json`) instead of re-pulling and re-unpacking the image — faster and avoids duplicate work in the same pipeline.

## Common Mistakes

- Generating SBOMs but never storing or attaching them anywhere durable — when a new CVE drops, there's nothing to query, so the SBOM step becomes a compliance checkbox that provides zero actual incident-response value.
- Picking SPDX or CycloneDX arbitrarily per-team, leading to a fleet where half your images have one format and half have the other — tooling that only understands one format silently produces incomplete results across the fleet.
- Assuming an SBOM alone tells you about vulnerabilities — it's an inventory, not a vulnerability report. You still need a scanner to cross-reference the SBOM's package list against a CVE database.
- Not re-generating the SBOM when the Dockerfile changes but the version tag doesn't (e.g., `:latest` rebuilt with new base image patches) — the attached SBOM silently goes stale and no longer reflects what's actually running.
- Skipping SBOM generation for "internal-only" images because "we're not selling to the government" — ignoring that internal incident-response speed during the next Log4Shell-style event is the bigger practical payoff, independent of any compliance mandate.
