# Day 12 — Container Security: Image Scanning

**Phase:** 0 – Foundation | **Week:** W2 | **Domain:** Containers | **Flag:** ⚡ Interview-critical

## Brief

Runtime hardening (previous file) limits what a compromised container can *do*. Image scanning is about not shipping a compromised container in the first place — finding known-vulnerable packages baked into your image layers before they reach production. It's the container-world equivalent of dependency auditing (`npm audit`, `pip-audit`), extended to cover OS packages, not just application dependencies, because a container image is really "a slice of a Linux distro plus your app" and both halves carry CVE risk.

## How vulnerability scanners actually work

A scanner doesn't run your code or look for behavior — it's fundamentally an inventory-and-match problem:

1. **Walk the image layers** and extract every installed package's name and exact version — from OS package databases (`dpkg`'s status file on Debian/Ubuntu images, the `rpm` database on RHEL/Fedora-based images, `apk`'s installed-packages db on Alpine) and from language-ecosystem manifests baked into the image (`package-lock.json`, `requirements.txt`/`poetry.lock`, `go.sum`, `Gemfile.lock`, JAR/`pom.xml` metadata).
2. **Match that inventory against vulnerability databases** — the NVD (National Vulnerability Database, the canonical CVE source), plus distro-specific security trackers (Debian Security Tracker, Red Hat's OVAL/RHSA feeds, Alpine's `secdb`, Ubuntu's USN feed) and the GitHub Advisory Database for language package ecosystems.
3. **Compare version ranges** — a CVE entry says "affected in versions ≥ a, < b"; the scanner checks whether the installed version falls in that range, and reports a hit with a severity (derived from the CVSS score, bucketed into LOW/MEDIUM/HIGH/CRITICAL — bucket cutoffs differ slightly between scanner vendors).

Two things follow from this mechanism that matter operationally. First, scanning is inherently **reactive** — it can only flag CVEs that have already been published; a zero-day sitting in a dependency produces no hit until someone discovers and discloses it. Second, **distro-patched-but-version-unchanged** situations are a real source of false positives: Debian frequently backports a security fix into a package without bumping its upstream version string, so a naive version-range check flags it as vulnerable when it's actually patched. Good scanners consume distro-specific patch metadata (not just upstream version numbers) specifically to avoid this — it's one differentiator between scanner quality.

## Trivy

Aqua Security's open-source scanner — a single static binary with no runtime dependencies, which makes it trivial to drop into any CI system. It scans OS packages, language dependencies, Infrastructure-as-Code misconfigurations (Dockerfile, Kubernetes manifests, Terraform), and accidentally-baked-in secrets, and can also emit an SBOM.

```bash
trivy image myapp:latest                                   # full scan, all severities
trivy image --severity HIGH,CRITICAL myapp:latest           # filter noise, focus on what matters
trivy image --severity HIGH,CRITICAL --exit-code 1 myapp:latest   # CI gate: nonzero exit fails the build
trivy image --ignore-unfixed myapp:latest                   # hide CVEs with no available fix — nothing actionable yet
trivy fs .                                                   # scan a filesystem/repo directly, no image build needed
trivy config Dockerfile                                      # IaC misconfig scan of the Dockerfile itself
```

`--exit-code 1` combined with `--severity` is the standard CI-gating pattern: Trivy exits `0` (pass) unless a finding at or above the given severity exists, in which case it exits nonzero and the pipeline step fails. Without `--exit-code`, Trivy always exits `0` regardless of findings — it's purely a reporting tool until you opt into gating.

Trivy maintains a local cache of its vulnerability database and refreshes it on each run by default; `--skip-db-update` avoids that refresh, useful in airgapped environments or to avoid rate-limiting when many CI jobs run in parallel.

## Grype

Anchore's open-source scanner, built around a two-step pipeline: generate a Software Bill of Materials (SBOM) first with its sibling tool **Syft**, then match that SBOM against vulnerability feeds.

```bash
syft myapp:latest -o json > sbom.json
grype sbom:sbom.json
grype myapp:latest                # convenience: runs Syft internally, then scans
grype myapp:latest --fail-on high
```

The SBOM-first design decouples "what's in this image" from "what's vulnerable right now" — generate the SBOM once at build time, store it as an artifact, and re-run vulnerability matching against it later (with Grype, or a different scanner entirely) without touching the image again. This matters in supply-chain-conscious pipelines where the SBOM itself becomes something you store, attach to the image, and potentially sign (see the next file) — the scan result is a snapshot in time, but the SBOM is a durable record of exactly what's inside.

## Docker Scout

Docker's native scanning, built into Docker Desktop, Docker Hub, and the `docker scout` CLI plugin.

```bash
docker scout cves myapp:latest
docker scout cves --only-severity critical,high myapp:latest
docker scout compare myapp:latest --to myapp:previous       # diff CVEs between two versions/tags
docker scout recommendations myapp:latest                    # suggests base-image changes that reduce CVE count
```

The standout feature is `recommendations` — instead of just reporting what's wrong, it suggests smaller or more recently patched base image alternatives that would reduce the CVE count. That's a meaningfully different workflow from "here's a list of 40 CVEs, good luck" — it points directly at the highest-leverage fix, which is very often "your base image is stale, bump it," rather than chasing individual package CVEs one at a time.

## Why scanning has to happen at multiple points

A single scan at build time is not sufficient, because the set of known CVEs against a given package version only grows over time — an image that was completely clean the day it was built can be sitting on a disclosed CRITICAL CVE a month later, with nothing about the image itself having changed.

- **Build time** (CI, right after `docker build`) — catches known-vulnerable base images and dependencies before the image ever reaches a registry; fail the pipeline before merge.
- **Registry** (scan-on-push and scheduled re-scans of already-stored images) — registries like ECR, GHCR, Harbor, and Docker Hub can scan on push and periodically re-scan everything already stored. This is what catches a CVE disclosed *after* your image passed its original build-time scan.
- **Runtime** (admission-time checks or continuous scanning of what's actually deployed, e.g. via a Kubernetes admission controller or an in-cluster agent like the Trivy Operator) — catches drift where a workload that hasn't changed at all is now flagged, because the world's knowledge of its dependencies changed underneath it.

"We scan in CI" is a necessary but incomplete answer to "how do you manage container vulnerabilities" — the complete answer covers all three points, because each one catches a different failure mode the others miss.

## Points to Remember

- Scanning is inventory-and-match against known-CVE databases (NVD + distro advisories + language advisory databases) — it cannot catch zero-days, only already-disclosed vulnerabilities.
- `--severity` + `--exit-code` (Trivy) or `--fail-on` (Grype) is what turns a scanner from a reporting tool into an enforced CI gate — without it, findings are just printed and the pipeline continues regardless.
- Grype/Syft split "inventory the image" (SBOM) from "match against CVEs" into two reusable steps; Trivy does both in one pass by default but can also emit SBOMs.
- Docker Scout's differentiator is prescriptive remediation (base-image recommendations), not just detection.
- Scan at build time, at the registry, and at runtime — a clean build-time scan is a snapshot, not a permanent guarantee.
- `--ignore-unfixed` is useful to cut noise from CVEs with no available patch yet, but don't use it to silently hide things you should be tracking for when a fix does land.

## Common Mistakes

- Running scans in CI without `--exit-code`/`--fail-on`, so the pipeline "passes" regardless of what the scanner found — the scan becomes decorative.
- Scanning once at build time and treating the image as permanently clean, missing CVEs disclosed against it after the fact.
- Chasing every LOW/MEDIUM finding in a large legacy image instead of gating on HIGH/CRITICAL first — burns review time on low-impact findings while CRITICAL ones sit unresolved.
- Not distinguishing "vulnerable, no fix available yet" from "vulnerable, fix available, just not applied" — the former needs monitoring, the latter needs an immediate dependency bump.
- Assuming a scanner covers language dependencies by default without checking — some configurations/older versions of a given tool need an explicit flag or plugin to scan `node_modules`/`site-packages` contents rather than just OS packages.
- Ignoring IaC misconfiguration findings (Trivy's `config`/Dockerfile scanning) because "that's not a CVE" — a container running `--privileged` or with a world-writable secrets file is a misconfiguration with the same real-world impact as a CVE.
