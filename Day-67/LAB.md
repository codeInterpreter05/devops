# Day 67 — Lab: Container & Registry Security

**Goal:** Build a real pipeline stage that scans, signs, and generates an SBOM for a container image — then verify the signature at Kubernetes admission time, exactly as described in the day's core hands-on activity.

**Prerequisites:**
- Docker (or another OCI builder) installed locally.
- `trivy`, `cosign`, and `syft` installed (`brew install trivy cosign syft` on macOS, or download the release binaries — all three ship as static binaries for Linux too).
- A local Kubernetes cluster for the admission-control step (`kind` or `minikube`).
- (Optional but recommended) A GitHub repo you control, to exercise real OIDC keyless signing instead of local key-based signing.

---

### Lab 1 — Scan an image and enforce a CRITICAL-CVE gate

1. Build a deliberately old/vulnerable test image:
   ```bash
   mkdir -p /tmp/day67 && cd /tmp/day67
   cat > Dockerfile <<'EOF'
   FROM node:14-alpine
   COPY . /app
   WORKDIR /app
   EOF
   docker build -t day67-test:latest .
   ```
2. Scan it and observe findings:
   ```bash
   trivy image day67-test:latest
   ```
3. Now run it as a real CI gate — fail if any CRITICAL, fixable CVE is present:
   ```bash
   trivy image --severity CRITICAL --ignore-unfixed --exit-code 1 day67-test:latest; echo "Exit code: $?"
   ```
4. Add an `.trivyignore` for one CVE you've "triaged," with a comment explaining why, and re-run — confirm the exit code changes.

**Success criteria:** You can produce a Trivy command that reliably fails a CI job (non-zero exit) on fixable CRITICAL CVEs, and you understand how `.trivyignore` changes that result.

---

### Lab 2 — Sign the image and verify with Cosign

1. Generate a local key pair (simulating what keyless signing does via Fulcio, without needing a real OIDC provider):
   ```bash
   cosign generate-key-pair
   ```
2. Push the image to a local registry so it has a real digest to sign:
   ```bash
   docker run -d -p 5000:5000 --name registry registry:2
   docker tag day67-test:latest localhost:5000/day67-test:latest
   docker push localhost:5000/day67-test:latest
   DIGEST=$(docker inspect --format='{{index .RepoDigests 0}}' localhost:5000/day67-test:latest | cut -d@ -f2)
   echo $DIGEST
   ```
3. Sign it:
   ```bash
   cosign sign --key cosign.key localhost:5000/day67-test@$DIGEST --allow-insecure-registry
   ```
4. Verify it:
   ```bash
   cosign verify --key cosign.pub localhost:5000/day67-test@$DIGEST --allow-insecure-registry
   ```
5. **Break it on purpose**: re-tag and push a *different* image to the same tag, then try verifying the old digest still works but confirm the new tag (unsigned) fails verification — demonstrating why digest pinning matters.

**Success criteria:** You can sign an image by digest, verify it successfully, and explain why verifying by mutable tag instead of digest would be unsafe.

---

### Lab 3 — Generate an SBOM and cross-scan it

1. Generate an SBOM in CycloneDX format:
   ```bash
   syft day67-test:latest -o cyclonedx-json > sbom.cdx.json
   syft day67-test:latest -o table | head -30
   ```
2. Feed the SBOM into Grype instead of re-scanning the image directly:
   ```bash
   grype sbom:./sbom.cdx.json --fail-on critical
   ```
3. Count how many packages were catalogued and identify the 5 with the oldest/lowest version numbers — these are your best guesses for likely CVE sources.

**Success criteria:** You have a working SBOM file, and can explain the difference between what Syft (inventory) and Trivy/Grype (vulnerability findings) each produce.

---

### Lab 4 — The core hands-on activity: verify a signature via Kubernetes admission

1. Spin up a local cluster:
   ```bash
   kind create cluster --name day67
   ```
2. Install Sigstore's `policy-controller`:
   ```bash
   kubectl create namespace cosign-system
   helm repo add sigstore https://sigstore.github.io/helm-charts
   helm install policy-controller sigstore/policy-controller -n cosign-system
   ```
3. Apply a `ClusterImagePolicy` requiring your public key (adjust for the local key from Lab 2, or a real keyless identity if you pushed to a real registry via GitHub Actions):
   ```yaml
   apiVersion: policy.sigstore.dev/v1beta1
   kind: ClusterImagePolicy
   metadata:
     name: require-signed-day67
   spec:
     images:
       - glob: "**/day67-test*"
     authorities:
       - key:
           data: |
             <paste contents of cosign.pub here>
   ```
4. Try to run an **unsigned** image with the same name pattern — confirm the Pod is rejected at admission time (`kubectl describe pod` / the `kubectl run` error itself).
5. Run the signed image and confirm it schedules successfully.

**Success criteria:** You can demonstrate, live, that an unsigned image matching the policy's glob is rejected before it ever reaches a node, and a properly signed one is admitted — this is the concrete answer to the day's interview question.

---

### Cleanup

```bash
kind delete cluster --name day67
docker rm -f registry
rm -rf /tmp/day67 cosign.key cosign.pub sbom.cdx.json
```

### Stretch challenge

Wire Labs 1–3 into an actual GitHub Actions workflow (`.github/workflows/day67.yml`) using keyless signing (`permissions: id-token: write`, no local key files), so the whole scan → sign → SBOM chain runs on every push, and the resulting image's identity is verifiable with `cosign verify --certificate-identity=... --certificate-oidc-issuer=...` instead of a local public key file.
