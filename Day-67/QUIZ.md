# Day 67 — Quiz: Container & Registry Security

Try to answer without looking at your notes. Answers are at the bottom.

1. What's the difference between what a vulnerability scanner (Trivy/Grype) tells you and what image signing (Cosign) tells you? Why do you need both?
2. What CLI flag turns a Trivy scan from "informational output" into an actual CI gate?
3. Why would you use `--ignore-unfixed` when gating on CRITICAL CVEs, and what's the tradeoff?
4. What's the difference between ECR basic scanning and enhanced scanning — engine, coverage, and frequency?
5. Explain Sigstore's keyless signing flow: what role do Fulcio and Rekor each play, and why does this avoid needing a long-lived private key?
6. Why must you always sign and verify images by digest rather than by tag?
7. In a GitHub Actions workflow, what permission is required for keyless Cosign signing to work, and what happens if you omit it?
8. What are the two SBOM formats mentioned, and what's each one more associated with?
9. Why is it more efficient to run `grype sbom:./sbom.json` than `grype myapp:latest` when you've already generated an SBOM with Syft in the same pipeline?
10. What's the actual enforcement mechanism that prevents an unsigned image from running in a Kubernetes cluster — and why isn't "the CI pipeline signs images" alone sufficient enforcement?
11. Why does an SBOM matter for incident response, concretely, using the Log4Shell example?
12. **Interview question:** How do you ensure only signed, scanned images can run in your Kubernetes cluster?

---

## Answers

1. A scanner tells you whether *known vulnerabilities* exist in the packages inside an image, as of the scan's vulnerability database snapshot. Signing tells you *integrity and provenance* — that this exact image (by digest) was produced by a specific, trusted build process and hasn't been altered since. You need both because a "clean" scan says nothing about whether the image was later tampered with, and a valid signature says nothing about whether the image contains known CVEs.
2. `--exit-code 1` (combined with a `--severity` filter) makes Trivy exit non-zero when matching vulnerabilities are found, which is what fails the CI job. Without it, Trivy just prints a report and the pipeline continues regardless of findings.
3. `--ignore-unfixed` skips CVEs that have no vendor-published fix available yet. The tradeoff: you might miss awareness of a real, currently-unpatched risk if you don't track it separately, but without this flag, builds can get permanently blocked by CVEs no code or dependency change can currently resolve — which isn't actionable and leads teams to disable the gate entirely out of frustration.
4. Basic scanning uses the Clair engine, covers OS packages only, and is point-in-time (triggered on push). Enhanced scanning uses Amazon Inspector, covers OS packages *and* language-level dependencies (Python, Java, Node, Go, Ruby, .NET), and re-scans continuously as new CVEs are disclosed — not just at push time.
5. Fulcio is Sigstore's certificate authority: it issues a short-lived (minutes) signing certificate bound to a verified OIDC identity (e.g., a specific GitHub Actions workflow) instead of requiring a long-lived private key. Rekor is a public, append-only transparency log that records the signing event (identity + artifact digest + timestamp) so it's auditable later. Together, they replace "protect a secret key forever" with "prove your identity at signing time via OIDC, and let a public log record that it happened" — nothing long-lived to leak or rotate.
6. Tags (e.g., `:latest`, `:v1.2.3`) are mutable pointers — they can be re-pushed to point at entirely different image content at any time. A digest (`sha256:...`) is an immutable cryptographic hash of the actual content. Signing/verifying by tag means the signature could apply to content that's since been silently swapped; signing/verifying by digest guarantees you're checking the exact bytes that were signed.
7. `permissions: id-token: write` in the job/workflow. Without it, the job has no OIDC token to present to Fulcio, and keyless signing fails with an identity/authentication error even though everything else in the pipeline is configured correctly.
8. SPDX (more compliance/legal-oriented, common in regulated/government contexts) and CycloneDX (originated in the AppSec/OWASP world, more oriented toward vulnerability-management tooling).
9. Because Grype can read the already-generated SBOM's package list directly instead of re-pulling and re-unpacking the full image to build its own inventory from scratch — saving time and avoiding duplicate work for something already computed once in the pipeline.
10. An admission-control webhook (Sigstore's `policy-controller`, or Kyverno's `verifyImages` rule) that checks image signatures against a trusted identity *before* allowing a Pod to schedule. "CI signs images" alone is not enforcement — nothing stops someone from deploying an unsigned or differently-sourced image directly to the cluster unless something at the cluster boundary actually rejects it.
11. An SBOM is a pre-computed inventory of every package (with version) in every image you've shipped. When a new CVE (like Log4Shell) is disclosed, you can query your stored SBOMs directly ("which images contain `log4j-core < 2.17`?") instead of re-scanning every image you've ever built — turning a multi-day audit into a fast, targeted query, but only if SBOMs were generated and stored ahead of time.
12. Strong answer: "Three layers working together: (1) CI scans every image with Trivy/Grype and fails the build on fixable CRITICAL CVEs before the image is ever pushed; (2) CI signs the resulting image by digest with Cosign, using keyless signing tied to the specific CI workflow's OIDC identity, and attaches a Syft-generated SBOM as a signed attestation; (3) at the cluster boundary, an admission webhook — Sigstore's `policy-controller` or Kyverno's `verifyImages` — checks every Pod's image signature against that exact trusted identity before allowing it to schedule, rejecting anything unsigned or signed by an untrusted source. The registry-side scan (e.g., ECR enhanced scanning) is a defense-in-depth layer since new CVEs get disclosed after images are already pushed." Mention that signature verification alone doesn't guarantee "vulnerability-free" — it guarantees provenance/integrity — which is why both scanning and signing need to be enforced together, not either alone.
