# Day 10 — Lab: Docker Deep Dive I

**Goal:** Take a naive, bloated Dockerfile and refactor it into a multi-stage, distroless build — then *prove* the improvement with `dive` and `hadolint` instead of eyeballing it.

**Prerequisites:**
- Docker installed and running (`docker version` should show both a Client and Server section).
- `dive` installed: `brew install dive` (macOS), or download a release from [github.com/wagoodman/dive](https://github.com/wagoodman/dive). Verify: `dive --version`.
- `hadolint` installed: `brew install hadolint`, or run it via its own container (no install needed): `docker run --rm -i hadolint/hadolint < Dockerfile`. Verify: `hadolint --version`.

---

### Lab 1 — Build the naive baseline

This is deliberately the "first draft" a lot of real Dockerfiles look like.

1. Set up the project:
   ```bash
   mkdir -p ~/labs/day10/naive-app && cd ~/labs/day10/naive-app
   ```
2. Create `app.py`:
   ```python
   from flask import Flask

   app = Flask(__name__)

   @app.route("/")
   def index():
       return {"status": "ok"}

   if __name__ == "__main__":
       app.run(host="0.0.0.0", port=5000)
   ```
3. Create `requirements.txt`:
   ```
   flask==3.0.3
   ```
4. Create `Dockerfile`:
   ```dockerfile
   FROM python:3.11
   WORKDIR /app
   COPY . .
   RUN pip install -r requirements.txt
   EXPOSE 5000
   CMD ["python", "app.py"]
   ```
5. Build and measure:
   ```bash
   docker build -t day10-naive:v1 .
   docker images day10-naive
   ```

**Success criteria:** Note the reported size — on the full `python:3.11` base this typically lands in the ~1.0–1.2GB range. Run `docker run --rm -p 5000:5000 day10-naive:v1` and confirm `curl localhost:5000` returns `{"status":"ok"}` before moving on — the refactor in Lab 2 needs to preserve this behavior, not just shrink the image.

---

### Lab 2 — The core hands-on activity: refactor to multi-stage distroless

This is the assigned hands-on activity for today — do the refactor for real, then measure it, don't just read the before/after.

1. Set up a second project directory reusing the same `app.py` and `requirements.txt`:
   ```bash
   mkdir -p ~/labs/day10/distroless-app && cd ~/labs/day10/distroless-app
   cp ../naive-app/app.py ../naive-app/requirements.txt .
   ```
2. Create the refactored `Dockerfile`:
   ```dockerfile
   # syntax=docker/dockerfile:1
   FROM python:3.11-slim AS builder
   WORKDIR /app
   COPY requirements.txt .
   RUN pip install --user --no-cache-dir -r requirements.txt

   FROM gcr.io/distroless/python3-debian12
   WORKDIR /app
   COPY --from=builder /root/.local /root/.local
   COPY app.py .
   ENV PATH=/root/.local/bin:$PATH
   EXPOSE 5000
   CMD ["app.py"]
   ```
3. Build and measure:
   ```bash
   docker build -t day10-distroless:v1 .
   docker images | grep day10
   ```
4. Confirm it still runs correctly:
   ```bash
   docker run --rm -p 5000:5000 day10-distroless:v1
   # in another terminal:
   curl localhost:5000
   ```
5. Confirm there's no shell (this is expected, not a bug):
   ```bash
   docker exec -it $(docker ps -q --filter ancestor=day10-distroless:v1) sh
   # should fail — distroless has no /bin/sh
   ```

**Success criteria:** `day10-distroless:v1` reports well under 100MB (typically in the tens of MB for Flask on distroless Python) against `day10-naive:v1`'s ~1GB+ — at least a 10x reduction — while `curl localhost:5000` returns the identical response from both images.

---

### Lab 3 — Measure the difference with `dive`

1. Inspect the naive image:
   ```bash
   dive day10-naive:v1
   ```
   In the TUI: `Tab` to switch between the layer list and the file tree, arrow keys to step through layers, and note the efficiency score and wasted-space figure at the bottom.
2. Inspect the distroless image the same way:
   ```bash
   dive day10-distroless:v1
   ```
3. Compare the two: which layers in the naive image are contributing the most bytes? (You should see the base `python:3.11` layer itself dwarfing everything else.)
4. Run `dive` in CI mode against both images and compare the pass/fail output:
   ```bash
   CI=true dive day10-naive:v1
   CI=true dive day10-distroless:v1
   ```

**Success criteria:** You can point to a specific layer in `day10-naive:v1` responsible for the bulk of its size (the base image layer), and state the wasted-space number `dive` reports for each image.

---

### Lab 4 — Lint both Dockerfiles with `hadolint`

1. Lint the naive Dockerfile:
   ```bash
   hadolint ~/labs/day10/naive-app/Dockerfile
   ```
   Expect hits similar to: an unpinned `pip install` (no `--no-cache-dir`, no hash pinning), and possibly a missing pinned tag warning depending on your hadolint ruleset version.
2. Lint the refactored Dockerfile:
   ```bash
   hadolint ~/labs/day10/distroless-app/Dockerfile
   ```
3. For every genuine finding, fix the Dockerfile and re-run `hadolint` until it's clean, or explicitly suppress a rule with a one-line justification comment above the instruction and `--ignore DL####` if the rule truly doesn't apply.
4. Confirm exit codes explicitly (0 = clean):
   ```bash
   hadolint ~/labs/day10/distroless-app/Dockerfile; echo "exit code: $?"
   ```

**Success criteria:** Both Dockerfiles either pass `hadolint` cleanly or have every remaining finding explicitly and deliberately suppressed with a stated reason — not silently ignored.

---

### Lab 5 — `.dockerignore` and cache-ordering check

1. In `~/labs/day10/naive-app`, add some noise a real repo would have:
   ```bash
   mkdir -p .git && dd if=/dev/urandom of=.git/fake-large-object bs=1M count=50 2>/dev/null
   ```
2. Rebuild without a `.dockerignore` and note the `[internal] load build context` step's reported size/duration in the build output:
   ```bash
   docker build -t day10-naive:v2 .
   ```
3. Add a `.dockerignore`:
   ```
   .git
   __pycache__/
   *.pyc
   .venv/
   Dockerfile
   .dockerignore
   ```
4. Rebuild and compare the build-context step again:
   ```bash
   docker build -t day10-naive:v3 .
   ```
5. Now check instruction ordering: touch `app.py` (change a comment only) and rebuild both the naive Dockerfile and the distroless one. Confirm that in the naive version `pip install` reruns every time (because `COPY . .` precedes it), while in the distroless version the `builder` stage's `pip install` layer stays cached (because `COPY requirements.txt .` precedes it and `requirements.txt` didn't change).

**Success criteria:** You can state, from direct observation of the build output (not just the file), which of the two Dockerfiles reinstalls dependencies on every source change and which doesn't, and why.

---

### Cleanup

```bash
docker rmi day10-naive:v1 day10-naive:v2 day10-naive:v3 day10-distroless:v1
docker system prune -f
```

---

### Stretch challenge

Pick one:

1. **BuildKit secret mount.** Create a fake credential for lab purposes only — never a real key:
   ```bash
   cd ~/labs/day10/distroless-app
   echo "fake-lab-api-key-do-not-use-000111" > secret.txt
   printf 'secret.txt\n' >> .gitignore
   printf 'secret.txt\n' >> .dockerignore
   ```
   Add a step to the `builder` stage that "uses" the secret without ever writing it to a layer:
   ```dockerfile
   RUN --mount=type=secret,id=api_key \
       test -s /run/secrets/api_key && echo "secret mounted, length: $(wc -c < /run/secrets/api_key)"
   ```
   Build with the secret supplied out-of-band, then prove it never landed in the image:
   ```bash
   DOCKER_BUILDKIT=1 docker build --secret id=api_key,src=./secret.txt -t day10-distroless:secret .
   docker history --no-trunc day10-distroless:secret | grep -i "fake-lab-api-key" || echo "not found in history — good"
   ```

2. **Shrink further with `scratch`.** Write a minimal static Go HTTP server, build it multi-stage with `CGO_ENABLED=0`, and ship it on `scratch` instead of distroless (see 01-README-Multi-Stage-Builds.md's Go example). Measure with `docker images` and `dive` — this should land in the low single-digit MB, smaller than even the distroless Python image, since there's no interpreter to ship at all.
