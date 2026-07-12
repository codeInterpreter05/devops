# Day 10 — Docker Deep Dive I: Multi-Stage Builds

**Phase:** 0 – Foundation | **Week:** W2 | **Domain:** Containers | **Flag:** ⚡ Interview-critical

## Brief

A Dockerfile that "just works" is easy to write and easy to get badly wrong in a way that only shows up as a number: `docker images` reporting 1.2GB for an app whose actual code is a few kilobytes. That gap is pure waste — slower pulls, slower deploys, a bigger attack surface, more CVEs for your scanner to flag — and it's caused almost entirely by one habit: copying your whole build environment into your runtime environment. Today is about closing that gap with multi-stage builds, distroless images, and `scratch`, and about the caching/tooling discipline (covered in the next two files) that makes rebuilding those images fast instead of painful.

This day is split into three focused files:

1. **This file** — why naive images bloat, multi-stage build syntax, distroless and `scratch`, worked Python and Go examples.
2. **[02-README-Layer-Caching-And-Build-Context.md](02-README-Layer-Caching-And-Build-Context.md)** — how the layer cache actually invalidates, instruction ordering, `.dockerignore`, and using `dive` to find bloat.
3. **[03-README-BuildKit-Secrets-And-Hadolint.md](03-README-BuildKit-Secrets-And-Hadolint.md)** — why `ARG`/`ENV` leak secrets into image history, BuildKit's `RUN --mount=type=secret`, and linting Dockerfiles with `hadolint`.

## Why naive Dockerfiles bloat

A typical first-pass Dockerfile for a Python service looks like this:

```dockerfile
FROM python:3.11
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
CMD ["python", "app.py"]
```

Every one of Docker's image layers is **additive and permanent** within that image — nothing you do in a later layer removes bytes a previous layer already committed to the image's shipped filesystem (it can only hide them behind a whiteout, still present in the layer blob). Walk through what actually ends up in the final image here:

- **The full `python:3.11` base** — not the `-slim` or `-alpine` variant — which is Debian plus the complete CPython build toolchain, `pip`, header files, and common libraries. This alone is commonly 900MB+.
- **Your entire build context** via `COPY . .` — including `.git` (which can be hundreds of megabytes of history), local virtualenvs, test fixtures, `__pycache__`, notebooks, and anything else sitting in the repo that a missing `.dockerignore` didn't filter out (see file 2).
- **`pip`'s wheel/build cache** and any compiler toolchain `pip` pulled in transiently to build a wheel from source for a package without a prebuilt wheel for your platform — those build artifacts (and sometimes `gcc` itself, if installed ad hoc) sit in the image forever unless you explicitly strip them in the *same* layer.
- **Source code you don't need at runtime** — tests, docs, CI config, `.md` files — none of it is excluded just because your app doesn't import it.

Add those up and 1.2GB for what's functionally a Flask app plus three dependencies is normal, not an outlier. None of this is a runtime problem in the sense that the app runs fine — it's a **shipping** problem: bigger pulls on every deploy and every autoscaled node, more surface area for a vulnerability scanner (and an attacker) to find in packages you never actually use at runtime, and slower CI.

## Multi-stage build syntax

A multi-stage build lets you use one (or more) throwaway image as your **build environment**, and a separate, minimal image as your **runtime environment** — only explicitly copying the compiled/installed artifacts across, never the toolchain that produced them.

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
CMD ["app.py"]
```

Mechanics that matter:

- Each `FROM` starts a **new, independent stage**. `AS builder` gives that stage a name you can reference later; without a name, stages are addressed by index (`--from=0`).
- `COPY --from=builder <path> <path>` is the only bridge between stages — it copies specific files out of a previous stage's filesystem into the current one. Nothing else about the builder stage (its base OS, its `pip` cache, `gcc`, source files you didn't `COPY --from`) makes it into the final image.
- You can have as many stages as you want, including copying from an image you didn't build at all: `COPY --from=nginx:1.27-alpine /etc/nginx/nginx.conf ./nginx.conf.default` pulls a single file out of a published image without ever running it.
- `docker build --target builder -t myapp:debug .` stops the build at a named intermediate stage — useful for getting a shell into the *build* environment to debug a failing dependency install, without needing a whole separate Dockerfile.
- Only the **final stage in the file** becomes the image that gets tagged and shipped by a plain `docker build`. Everything before it is cache-only unless you `--target` it directly.

## Caching across stages

Each stage is cached independently, using the same layer-cache rules as a single-stage build (full mechanics in file 2). This is one of the underrated wins of multi-stage builds: if you only change application code and not `requirements.txt`, the `builder` stage's `pip install` layer is served straight from cache — Docker doesn't reinstall dependencies just because the final stage's `COPY app.py .` changed. Structuring the builder stage so dependency installation happens *before* copying application source (as above) is what makes that caching payoff real; get the order backwards and you lose it.

## Distroless images

**Distroless** (Google's [`GoogleContainerTools/distroless`](https://github.com/GoogleContainerTools/distroless) project, published as `gcr.io/distroless/*`) images contain *only* your application and its runtime dependencies — no shell (`/bin/sh` doesn't exist), no package manager, no coreutils (`ls`, `cat`, `grep` are absent), and none of the other userland that a full distro ships "just in case." Common variants:

- `gcr.io/distroless/static-debian12` — for statically linked binaries (Go with `CGO_ENABLED=0`, Rust with a musl target). Contains only CA certs, `/etc/passwd`, and timezone data — nothing else.
- `gcr.io/distroless/base-debian12` — adds glibc and libssl for dynamically linked binaries that need a C runtime but not a language runtime.
- `gcr.io/distroless/python3-debian12`, `.../nodejs20-debian12`, `.../java17-debian12` — include the respective language runtime and its default `ENTRYPOINT` (e.g., distroless Python's entrypoint is `python3`, so a `CMD ["app.py"]` in your own Dockerfile becomes `python3 app.py`, exactly as in the example above).
- Every image also ships a `:nonroot` and `:debug` tag variant — `:nonroot` runs as an unprivileged UID by default (defense in depth if the app is ever compromised), and `:debug` includes a minimal BusyBox shell for exactly the situation where you need to `exec` in and there normally isn't a shell to exec into.

The tradeoff is debuggability: `docker exec -it <container> sh` simply fails against a non-`:debug` distroless container, because there's no shell binary to run. In practice you either keep a `:debug`-tagged image around for troubleshooting, or use an ephemeral debug container attached to the same process/network namespace (`docker debug <container>`, or `kubectl debug` in Kubernetes) rather than trying to shell into the distroless container directly.

## `scratch` — the empty base

`scratch` isn't a downloaded image at all — it's Docker's reserved keyword for **an empty filesystem**, zero bytes, not even a libc. It only works for a binary with **no dynamic dependencies whatsoever**: no libc, no shared libraries, no shell to invoke, nothing that expects `/etc/passwd` or `/etc/resolv.conf` to exist unless you copy those in yourself. This is realistic for Go (build with `CGO_ENABLED=0` to force static linking — cgo pulls in dynamic glibc linking on Linux by default whenever a dependency touches `net`, `os/user`, or similar) and for Rust targeting `x86_64-unknown-linux-musl`. It is not viable for a normal Python, Node, or Java app — those runtimes are dynamically linked interpreters/VMs with dozens of supporting files; `scratch` for them just produces a container that fails to start.

## Worked example: Python (naive vs. multi-stage distroless)

**Naive** (~1GB+, full `python:3.11`, `pip` cache, everything in the build context):

```dockerfile
FROM python:3.11
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
CMD ["python", "app.py"]
```

**Refactored** (final stage is distroless — no shell, no `pip`, no compiler, just the interpreter and installed packages):

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
CMD ["app.py"]
```

The builder stage still uses `-slim`, not distroless — it needs `pip` and (transiently) a compiler for any package without a prebuilt wheel, and none of that toolchain survives into the final stage. This is the exact refactor exercised in today's lab.

## Worked example: Go (naive vs. scratch)

**Naive** (~900MB+, ships the entire Go toolchain and source tree):

```dockerfile
FROM golang:1.22
WORKDIR /app
COPY . .
RUN go build -o server .
CMD ["./server"]
```

**Refactored** (a few MB — just the static binary plus CA certs for outbound HTTPS):

```dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.22 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o server .

FROM scratch
COPY --from=builder /app/server /server
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/ca-certificates.crt
ENTRYPOINT ["/server"]
```

`CGO_ENABLED=0` forces a static binary (no dynamic libc dependency); `-ldflags="-s -w"` strips debug symbols to shrink the binary further. The CA certificate bundle is copied explicitly because `scratch` has none — without it, any outbound TLS call from the binary fails with a certificate-verification error even though the binary itself runs fine.

## Points to Remember

- Multi-stage builds don't make the *builder* stage smaller — they make the builder stage **irrelevant to final image size**. Only what you `COPY --from=` survives.
- `docker build --target <stage>` builds and tags just one intermediate stage — useful for debugging a failing build step without a second Dockerfile.
- Distroless removes the shell and package manager, not your runtime — `gcr.io/distroless/python3-*` still runs Python fine, it just can't be shelled into without the `:debug` tag.
- `scratch` is only viable for statically linked binaries with zero dynamic dependencies — that means `CGO_ENABLED=0` for Go, and manually copying anything the binary implicitly expects (CA certs, timezone data, `/etc/passwd` if it does UID lookups).
- Caching still works per-stage: order each stage's own instructions least-to-most frequently changing (see file 2), the same rule doesn't stop applying just because you added stages.

## Common Mistakes

- Forgetting `COPY --from=builder` for a required file and getting a container that fails at startup with a confusing "file not found" — always verify the final stage actually contains everything the runtime needs (`docker run --rm -it <image>` against a `:debug`/non-distroless variant while iterating, then swap to distroless once it's validated).
- Using `scratch` or a fully static distroless variant for a dynamically linked binary — the failure mode is a cryptic `exec format error` or `no such file or directory` even though the file is clearly present, because the missing piece is the dynamic linker (`/lib64/ld-linux...`) or libc, not the binary itself.
- Building a Go binary without `CGO_ENABLED=0` and being surprised it dynamically links glibc anyway — cgo is enabled by default and gets pulled in transitively by common packages, silently reintroducing a dynamic dependency `scratch` can't satisfy.
- Skipping the CA certificate copy into a `scratch` image, then debugging a confusing TLS handshake/certificate error that has nothing to do with the application code.
- Treating distroless as a security *panacea* rather than one layer — it reduces attack surface (fewer binaries an attacker can pivot to if they get code execution) but doesn't replace vulnerability scanning, dependency pinning, or running as non-root (`:nonroot` tag).
