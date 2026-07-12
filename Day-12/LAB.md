# Day 12 — Lab: Container Security

**Goal:** Take a container image through the full security lifecycle — build something realistically vulnerable (standing in for "an internship image nobody's touched in a year"), scan it, fix the worst findings, harden how it runs, and sign it — so each concept from today's three README files becomes something you've actually done, not just read about.

**Prerequisites:**
- Docker installed and running.
- Trivy: `brew install trivy` (macOS) or see `https://aquasecurity.github.io/trivy/latest/getting-started/installation/` for your distro.
- Cosign: `brew install cosign` or download a release binary from `https://github.com/sigstore/cosign/releases`.
- Optional for Lab 4: Grype (`brew install grype`) and the `docker scout` CLI plugin (bundled with recent Docker Desktop; `docker scout version` to confirm).
- A scratch directory: `mkdir -p ~/labs/day12-container-security && cd ~/labs/day12-container-security`

---

### Lab 1 — Build your "internship image" (intentionally outdated)

You may not have a real internship image handy, so build a stand-in that reproduces the same problem: an old base image with outdated dependencies that nobody's rebuilt in a while.

1. Create `app.py`:
   ```python
   from flask import Flask

   app = Flask(__name__)

   @app.route("/")
   def index():
       return "hello from an intentionally outdated image\n"

   if __name__ == "__main__":
       app.run(host="0.0.0.0", port=5000)
   ```
2. Create `requirements-vulnerable.txt` pinning deliberately old, CVE-bearing versions:
   ```
   flask==1.0
   requests==2.19.1
   ```
3. Create `Dockerfile.vulnerable`:
   ```dockerfile
   FROM python:3.9-slim-buster

   WORKDIR /app
   COPY requirements-vulnerable.txt .
   RUN pip install --no-cache-dir -r requirements-vulnerable.txt
   COPY app.py .

   CMD ["python", "app.py"]
   ```
   Notice everything about this Dockerfile: no `USER` (runs as root), an EOL base image tag (`buster` is past end-of-life), no pinned patch-level base digest, old dependency versions. This is deliberately how a lot of real "it's been running fine for two years" images look.
4. Build and tag it:
   ```bash
   docker build -f Dockerfile.vulnerable -t day12-app:vulnerable .
   ```

**Success criteria:** `docker run --rm -p 5000:5000 day12-app:vulnerable` serves the hello-world response at `http://localhost:5000`, and you understand *why* each of the four things noted in step 3 is a smell, not just that they're present.

---

### Lab 2 — Scan it with Trivy, triage the top CVEs

1. Run a full scan first, unfiltered, to see the real volume of findings:
   ```bash
   trivy image day12-app:vulnerable
   ```
2. Cut the noise to what actually matters:
   ```bash
   trivy image --severity HIGH,CRITICAL day12-app:vulnerable
   ```
3. Confirm this is CI-gateable — run it as a gate and watch it fail:
   ```bash
   trivy image --severity HIGH,CRITICAL --exit-code 1 day12-app:vulnerable; echo "exit code: $?"
   ```
   You should see a nonzero exit code — this is exactly the check a CI pipeline would run to block a merge or a deploy.
4. From the HIGH/CRITICAL output, pick the **top 3 findings** to fix. Prioritize by: (a) CRITICAL severity first, (b) a "Fixed Version" is actually listed (no point prioritizing a CVE with no available patch yet), (c) it's in a package your app directly touches (Flask/requests) rather than a transitive OS library you're unlikely to reason about further. Write down each CVE ID, the affected package + installed version, and the fixed version Trivy reports.

**Success criteria:** You have a written list of 3 concrete CVE IDs with their affected package, installed version, and target fixed version — not just "there were a lot of findings."

---

### Lab 3 — Fix the top 3 CVEs

1. Create `requirements-fixed.txt` bumping the packages you identified to at least their fixed versions:
   ```
   flask==3.0.3
   requests==2.32.3
   ```
2. Create `Dockerfile.fixed-base` — same as before but on a current, non-EOL base image:
   ```dockerfile
   FROM python:3.12-slim

   WORKDIR /app
   COPY requirements-fixed.txt .
   RUN pip install --no-cache-dir -r requirements-fixed.txt
   COPY app.py .

   CMD ["python", "app.py"]
   ```
3. Build and re-scan:
   ```bash
   docker build -f Dockerfile.fixed-base -t day12-app:fixed .
   trivy image --severity HIGH,CRITICAL day12-app:fixed
   ```
4. Compare the two scans side by side:
   ```bash
   trivy image --severity HIGH,CRITICAL --format json day12-app:vulnerable > /tmp/vuln.json
   trivy image --severity HIGH,CRITICAL --format json day12-app:fixed > /tmp/fixed.json
   diff <(jq '.Results[].Vulnerabilities[]?.VulnerabilityID' /tmp/vuln.json | sort -u) \
        <(jq '.Results[].Vulnerabilities[]?.VulnerabilityID' /tmp/fixed.json | sort -u)
   ```
   The three CVE IDs you targeted should now be missing from the fixed image's output (`<` lines in the diff with no matching `>` line).

**Success criteria:** `trivy image --severity HIGH,CRITICAL --exit-code 1 day12-app:fixed` either exits `0`, or exits nonzero only for findings you've consciously decided to accept (e.g., no fix available yet) — not the three you set out to fix.

---

### Lab 4 — Cross-check with Grype and Docker Scout

Different scanners use different CVE feeds and matching logic, so results won't be identical — that's expected, not a bug in either tool.

1. Scan the fixed image with Grype:
   ```bash
   grype day12-app:fixed
   grype day12-app:fixed --fail-on high
   ```
2. Scan it with Docker Scout and pull a remediation suggestion:
   ```bash
   docker scout cves --only-severity critical,high day12-app:fixed
   docker scout recommendations day12-app:fixed
   docker scout compare day12-app:fixed --to day12-app:vulnerable
   ```

**Success criteria:** You can name at least one finding Grype or Trivy reported that the other didn't (or confirm they matched), and you've read Docker Scout's base-image recommendation output at least once, even if you don't act on it further today.

---

### Lab 5 — Harden the Dockerfile and how it's run

1. Create `Dockerfile.hardened`:
   ```dockerfile
   FROM python:3.12-slim

   RUN groupadd --gid 1000 appgroup \
    && useradd --uid 1000 --gid appgroup --shell /usr/sbin/nologin --create-home appuser

   WORKDIR /app
   COPY --chown=appuser:appgroup requirements-fixed.txt .
   RUN pip install --no-cache-dir -r requirements-fixed.txt
   COPY --chown=appuser:appgroup app.py .

   USER appuser
   EXPOSE 5000
   CMD ["python", "app.py"]
   ```
2. Build it:
   ```bash
   docker build -f Dockerfile.hardened -t day12-app:hardened .
   ```
3. Run it hardened — non-root, read-only root filesystem, capabilities dropped to nothing, seccomp default (implicit):
   ```bash
   docker run --rm -d --name day12-hardened \
     -p 5000:5000 \
     --read-only \
     --tmpfs /tmp \
     --cap-drop ALL \
     --user 1000:1000 \
     day12-app:hardened
   ```
4. Verify the hardening actually took effect:
   ```bash
   docker exec day12-hardened whoami        # should print "appuser" or fail — never "root"
   docker exec day12-hardened id -u          # should print 1000
   docker exec day12-hardened touch /new-file  # must fail: Read-only file system
   docker exec day12-hardened touch /tmp/scratch  # must succeed: tmpfs is writable
   grep Seccomp /proc/$(docker inspect --format '{{.State.Pid}}' day12-hardened)/status
   # Seccomp: 2  -> filtering active (Docker's default profile)
   curl -s http://localhost:5000/                # app still works despite all the restrictions
   ```
5. Confirm capabilities really are empty:
   ```bash
   docker inspect day12-hardened --format '{{.HostConfig.CapDrop}} {{.HostConfig.CapAdd}}'
   ```

**Success criteria:** The app still serves traffic correctly, `whoami`/`id -u` confirm non-root, an attempted write outside `/tmp` fails with a read-only filesystem error, and `CapDrop` shows `[ALL]` with `CapAdd` empty.

---

### Lab 6 — Sign the image with Cosign

Cosign needs the image to live in a registry (it stores the signature as an OCI artifact next to the image) — spin up a throwaway local registry so you don't need real registry credentials for this lab.

1. Start a local registry and push the hardened image to it:
   ```bash
   docker run -d -p 5000:5000 --restart=always --name local-registry registry:2

   docker tag day12-app:hardened localhost:5000/day12-app:hardened
   docker push localhost:5000/day12-app:hardened
   ```
2. Grab the content digest you'll actually sign (always sign the digest, never the mutable tag):
   ```bash
   DIGEST=$(docker inspect --format='{{index .RepoDigests 0}}' localhost:5000/day12-app:hardened)
   echo "$DIGEST"
   ```
3. Generate a key pair (you'll be prompted for a passphrase — remember it, you'll need it to sign):
   ```bash
   cosign generate-key-pair
   # writes cosign.key (private) and cosign.pub (public) in the current directory
   ```
4. Sign the image by digest:
   ```bash
   cosign sign --key cosign.key --allow-insecure-registry "$DIGEST"
   ```
   (`--allow-insecure-registry` is only needed because this lab's throwaway registry runs over plain HTTP on localhost — a real registry uses TLS and doesn't need this flag.)
5. Verify the signature:
   ```bash
   cosign verify --key cosign.pub --allow-insecure-registry "$DIGEST"
   ```
   This should print the signature payload and a confirmation. Now prove verification actually *means* something by trying it against an unsigned image:
   ```bash
   docker tag day12-app:vulnerable localhost:5000/day12-app:unsigned
   docker push localhost:5000/day12-app:unsigned
   UNSIGNED_DIGEST=$(docker inspect --format='{{index .RepoDigests 0}}' localhost:5000/day12-app:unsigned)
   cosign verify --key cosign.pub --allow-insecure-registry "$UNSIGNED_DIGEST"
   # expect: verification failure — no signatures found
   ```

**Success criteria:** `cosign verify` succeeds against the signed digest and produces a clear, expected failure ("no matching signatures") against the unsigned image — you've proven to yourself that verification is discriminating, not a rubber stamp.

---

### Cleanup

```bash
docker rm -f day12-hardened local-registry 2>/dev/null
docker rmi day12-app:vulnerable day12-app:fixed day12-app:hardened \
  localhost:5000/day12-app:hardened localhost:5000/day12-app:unsigned 2>/dev/null
rm -f cosign.key cosign.pub
rm -rf ~/labs/day12-container-security
```

### Stretch challenge

Pick one:

1. **CI gate:** Add a GitHub Actions workflow step that builds `Dockerfile.hardened`, runs `trivy image --severity HIGH,CRITICAL --exit-code 1` against it, and confirm the job fails when you intentionally reintroduce an old package version, then passes again once you revert. This is the real-world version of what Lab 2 did manually.
2. **Custom seccomp profile:** Trace the actual syscalls `day12-app:hardened` makes under a real request (`strace -f -c docker exec ...`, or a tool like Inspektor Gadget's `traceloop`), build a minimal JSON seccomp profile that allow-lists only those syscalls plus deny-by-default for everything else, and run the container with `--security-opt seccomp=./custom-profile.json`. Confirm the app still serves traffic, then confirm a syscall you deliberately left out of the allowlist (e.g., attempt an operation requiring `mount`) is blocked.
