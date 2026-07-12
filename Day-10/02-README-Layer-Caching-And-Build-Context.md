# Day 10 — Docker Deep Dive I: Layer Caching and Build Context

**Phase:** 0 – Foundation | **Week:** W2 | **Domain:** Containers | **Flag:** ⚡ Interview-critical

## Brief

Multi-stage builds (file 1) fix *what ships*. This file fixes *how fast you can rebuild it* — the difference between a CI pipeline that reinstalls every dependency on every commit and one that only pays that cost when dependencies actually change. That difference comes down to three things: understanding exactly how Docker's layer cache invalidates, ordering your Dockerfile instructions to exploit it, and controlling what gets sent into the build in the first place via `.dockerignore`. The last section covers `dive`, the tool you reach for when an image is bigger than it should be and you need to see *why*, layer by layer.

## How the layer cache works

Every instruction in a Dockerfile (`FROM`, `RUN`, `COPY`, `ADD`, `ENV`, `WORKDIR`, ...) produces one layer, and Docker caches each layer keyed on the instruction plus its inputs:

- For `RUN`, the cache key is essentially the **literal command string** (and the parent layer's cache key). Docker does not know or care what the command actually changed on disk — `RUN echo "different every time: $(date)"` is *not* automatically cache-busted just because its output differs; it's still cached against the literal string in the Dockerfile unless you defeat that intentionally (e.g., with a changing `ARG`).
- For `COPY`/`ADD`, the cache key includes a **checksum of the actual files/directories being copied** (content and select metadata like permissions) — not their timestamps. Touching a file's mtime without changing its content does not invalidate a `COPY` layer's cache; changing its content or permissions does.
- The cache is a **chain, not independent per-instruction memoization**: the moment one instruction's cache misses, every instruction *after* it in that stage reruns from scratch, even if their own individual inputs are unchanged. There's no way to "skip ahead" back into cache once you've missed once in a stage.

This chain behavior is why instruction *order* is the single highest-leverage lever you have over build speed.

## Ordering instructions by change frequency

The classic mistake:

```dockerfile
FROM node:20-slim
WORKDIR /app
COPY . .
RUN npm install
CMD ["node", "server.js"]
```

`COPY . .` copies application source *and* `package.json` together in one layer. Change a single line of source code — anything, even a comment — and that `COPY` layer's checksum changes, which invalidates every layer after it, including `RUN npm install`. Every commit now reinstalls the entire dependency tree, even though dependencies didn't change.

The fix is ordering by **how often each input actually changes**, least-frequent first:

```dockerfile
FROM node:20-slim
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm install
COPY . .
CMD ["node", "server.js"]
```

Now a source-only change only invalidates the final `COPY . .` layer — the `npm install` layer stays cached because `package.json`/`package-lock.json` didn't change. The equivalent Python pattern:

```dockerfile
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
```

The same principle applies to `apt-get`: `RUN apt-get update && apt-get install -y <packages>` must be **one instruction**, not two separate `RUN` lines. If `update` and `install` are split across layers, Docker can serve a stale cached `apt-get update` layer (from days or weeks ago) while running a fresh `install`, silently attempting to install a package version, or a package, that no longer exists in the index that was actually fetched — a well-known source of "works locally, fails in CI" flakiness.

## `.dockerignore`

`.dockerignore` uses gitignore-like glob syntax and lives next to the Dockerfile (or build context root). It excludes matching paths from the **build context** — the tarball the Docker CLI sends to the daemon (or BuildKit) before the build even starts.

```
.git
.gitignore
__pycache__/
*.pyc
.venv/
node_modules/
dist/
build/
*.log
.env
.env.*
Dockerfile
.dockerignore
README.md
```

Two distinct problems this solves:

1. **Build context size and speed.** Docker tars up and streams the *entire* context directory to the daemon before evaluating a single instruction — even files no `COPY` will ever touch. A repo with a large `.git` history or an uncommitted `node_modules`/`.venv` can turn a multi-megabyte app into a multi-hundred-megabyte context transfer on *every single build*, independent of caching. With BuildKit this shows up as a `[internal] load build context` step with its own duration in the build output — a slow one is a `.dockerignore` smell.
2. **Cache stability and secret leakage.** A broad `COPY . .` checksums whatever's in the context — an IDE swap file, a stray `.env`, a local SQLite dev database, or scratch output changing on every run invalidates the cache (and, in the `.env` case, can bake a real secret straight into an image layer if it's ever copied in, exactly the failure mode covered in file 3 for `ARG`/`ENV`). Excluding these keeps the cache stable and keeps things that were never meant to leave your machine from ending up in a shipped image.

`!` negates a pattern to re-include something a broader pattern excluded, evaluated in file order — e.g., `logs/*` then `!logs/.gitkeep`.

## Using `dive` to find bloat

`dive` is a terminal UI for inspecting an image layer by layer — what each layer added, modified, or removed, and crucially, a computed **wasted space** figure: bytes present in the image that are duplicated across layers (e.g., the same file written, then overwritten, in two different `RUN` steps) or otherwise avoidable.

```bash
dive myapp:latest              # inspect an already-built image
dive build -t myapp:latest .   # build, then immediately inspect
CI=true dive myapp:latest      # non-interactive: pass/fail against efficiency thresholds
```

In the interactive TUI, `Tab` switches between the layer list and the current layer's filetree, arrow keys navigate, and `Ctrl+A`/`Ctrl+L` toggle between showing all files vs. only what changed in the selected layer. The bottom bar reports an efficiency score and total wasted bytes — the two numbers you're trying to drive toward 100% and 0 respectively. `CI=true` mode is what you wire into a pipeline: it evaluates against a `.dive-ci` YAML config (`rules: { lowestEfficiency: 0.95, highestWastedBytes: "20MB", highestUserWastedPercent: 0.10 }`) and exits non-zero if the image regresses past those thresholds, turning "don't let images creep back up in size" into an enforced CI gate rather than a habit someone eventually forgets.

## Points to Remember

- `RUN` is cached by the literal command text; `COPY`/`ADD` are cached by a checksum of the actual file contents being copied. Neither is timestamp-based.
- One cache miss invalidates every subsequent instruction in that stage — there's no partial credit, so put the most volatile instructions (application source) last.
- `apt-get update && apt-get install -y ...` must be a single `RUN` — splitting them risks installing against a stale cached package index.
- `.dockerignore` solves two separate problems: a smaller/faster build context, and a stable cache that can't be invalidated (or leaked) by files you never meant to include.
- `dive` gives you a number (wasted bytes, efficiency %) for "how bloated is this image," which is what you need before you can say a refactor measurably helped.

## Common Mistakes

- `COPY . .` before installing dependencies, then wondering why every CI run reinstalls the entire dependency tree regardless of what changed.
- No `.dockerignore` at all, silently shipping `.git` (and its full history) or a local `.env` into the build context on every build.
- Splitting `apt-get update` and `apt-get install` into separate `RUN` layers, hitting stale-index install failures that don't reproduce locally because the local cache happens to be fresh.
- Assuming `dive`'s job is done once you've looked at it once — not wiring `CI=true dive` with a `.dive-ci` config into the pipeline means image bloat creeps back in unnoticed over time.
- Using `ADD` out of habit where `COPY` is all that's needed — `ADD`'s extra behaviors (auto-extracting local tarballs, fetching remote URLs) are rarely what you want and can introduce surprising, unverified content into a layer.
