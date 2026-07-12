# Day 10 — Docker Deep Dive I: BuildKit Secrets and Hadolint

**Phase:** 0 – Foundation | **Week:** W2 | **Domain:** Containers | **Flag:** ⚡ Interview-critical

## Brief

Every Dockerfile instruction produces a layer, and layers are permanent, inspectable parts of the image you ship — a fact files 1 and 2 used to explain image bloat and caching. This file uses the same fact to explain a security bug that shows up constantly in real repos: passing build-time secrets (npm tokens, private registry credentials, API keys needed to fetch a private dependency) via `ARG`/`ENV`, believing that because the secret isn't in a file you `COPY`, it's safe. It isn't — and BuildKit's `--mount=type=secret` exists specifically to fix this. This file also covers `hadolint`, the linter that catches this and a long list of other Dockerfile anti-patterns before they ship.

## Why `ARG`/`ENV` are the wrong way to pass secrets

```dockerfile
FROM python:3.11-slim
ARG API_KEY
ENV API_KEY=$API_KEY
RUN curl -sf -H "Authorization: Bearer $API_KEY" https://internal.example.com/fetch-config -o config.json
```

This builds and works, but the secret is now permanently embedded in the image in **two independent, recoverable places**:

1. **Build history.** `docker history --no-trunc myimage` prints the full command string of every layer, including this `RUN` line — the `curl` invocation with the key inlined is right there in plain text, forever, for anyone who can pull the image.
2. **Image config.** `ENV` values become part of the image's config manifest, visible via `docker inspect myimage` (`.Config.Env`) — this persists even if a *later* instruction does `ENV API_KEY=` or `RUN unset API_KEY`, because that only affects the *runtime* environment of a container started from a later layer; it does not remove the value from the earlier layer where it was set. The earlier layer, key and all, still exists in the image and is still pullable.

This is a real, common vulnerability class (CWE-798, hardcoded credentials) — public registries have repeatedly been found hosting images with live AWS keys and database passwords recoverable this exact way, purely by running `docker history` or unpacking the image tarball, no exploit required. "I didn't commit the secret to git" is not the same as "the secret isn't in the image" — the image itself is a distribution artifact, and everyone who can pull it can extract anything ever set via `ARG`/`ENV`, or written into any layer, at any point in the build.

## BuildKit's `RUN --mount=type=secret`

BuildKit adds a `--mount` flag to `RUN` that mounts a secret as a file at a temporary path **only for the duration of that specific `RUN` step's process** — it is never written into the layer filesystem diff that gets committed, so it never appears in `docker history`, `docker inspect`, or the final image at all.

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.11-slim
WORKDIR /app
RUN --mount=type=secret,id=api_key \
    API_KEY=$(cat /run/secrets/api_key) && \
    curl -sf -H "Authorization: Bearer $API_KEY" https://internal.example.com/fetch-config -o config.json
```

Build it by pointing `--secret` at a local file that holds the value — never at a literal string, and never at a file you also `COPY` into the image:

```bash
DOCKER_BUILDKIT=1 docker build --secret id=api_key,src=./secrets/api_key.txt -t myapp .
```

The secret is mounted at `/run/secrets/api_key` by default (`,target=/some/other/path` overrides that) and exists only inside the mount namespace of that one `RUN` instruction's process tree. Once that instruction finishes, the mount is gone — there is no layer diff containing it to inspect later. Rules that keep this actually secure in practice:

- Never `echo`, `cat >`, or otherwise persist the secret's value to a file inside the image's writable layer within that same `RUN` — reading it into a shell variable and using it immediately (as above) is fine; writing it to disk defeats the entire point.
- Never `COPY` the secret file itself into the build context in a way a later instruction could pick up — keep it outside version control (`.gitignore`) and outside the Docker build context (`.dockerignore`) entirely; `--secret src=` reads it directly from the host filesystem path you give it, independent of the build context.
- A related BuildKit mount, `RUN --mount=type=ssh`, solves the same class of problem for SSH-agent forwarding — e.g., `go mod download` or `pip install` needing to authenticate against a private Git repository during the build without baking an SSH key into any layer.

## Enabling BuildKit

Two things must both be true for `--mount=` syntax to parse at all:

1. **BuildKit itself must be the active builder.** Docker Engine 23.0+ and current Docker Desktop default to BuildKit already; on older installations, set `DOCKER_BUILDKIT=1` in the environment before running `docker build`.
2. **The Dockerfile must declare the newer frontend syntax** with `# syntax=docker/dockerfile:1` as its very first line. The legacy builder's parser doesn't understand `--mount` at all — without this line (even with BuildKit enabled) you get a hard parse error on the `RUN --mount=...` instruction.

Both together — `DOCKER_BUILDKIT=1` (or a modern default-BuildKit install) *and* the `# syntax=` pragma — are required; either alone is not sufficient.

## `hadolint`

`hadolint` is a static analysis linter purpose-built for Dockerfiles: it runs Shellcheck against your `RUN` lines' shell content and applies its own rule set (prefixed `DL####`) for Dockerfile-specific anti-patterns. Install via `brew install hadolint`, or run it without installing anything using its own container:

```bash
docker run --rm -i hadolint/hadolint < Dockerfile
hadolint Dockerfile                       # if installed locally
hadolint --ignore DL3008 Dockerfile       # suppress a specific rule
```

Rules worth knowing cold:

- **DL3006** — always pin the `FROM` image to a specific tag, never `latest`, so a rebuild six months from now doesn't silently pull a different base.
- **DL3008** — pin `apt-get install` package versions (`apt-get install -y curl=7.88.1-10`) for reproducible builds, since an unpinned install can resolve to a different version on every build.
- **DL3009** — delete the apt package list cache (`rm -rf /var/lib/apt/lists/*`) in the *same* `RUN` as the install, otherwise the cache survives as its own layer, adding bloat with zero runtime value.
- **DL4006** — set `SHELL ["/bin/bash", "-o", "pipefail", "-c"]` when a `RUN` uses a pipe, so a failure in an earlier command of the pipeline (e.g., `curl ... | tar -xz`) doesn't get masked by the pipe's last command succeeding.
- **DL3025** — use JSON array (`exec`) form for `CMD`/`ENTRYPOINT` (`CMD ["python", "app.py"]`) rather than shell form (`CMD python app.py`), so signals like `SIGTERM` reach your process directly instead of an intermediary shell.

Wire `hadolint Dockerfile` into CI as a pre-build gate — it exits non-zero on any unignored violation, so a bad Dockerfile fails the pipeline before you ever spend time building and pushing it. Triage genuine violations by fixing them; for the rare rule that doesn't apply to your situation, suppress it explicitly and visibly (`--ignore DL3008` or a checked-in `.hadolint.yaml`) rather than ignoring the tool's output wholesale.

## Points to Remember

- `ARG`/`ENV` secrets leak two ways: the literal command string in `docker history`, and the value in the image's config (`docker inspect`) — neither is fixed by unsetting the value in a later instruction.
- `RUN --mount=type=secret` mounts the secret only for that instruction's process, never writes it to a layer, and never shows up in `docker history` or `docker inspect`.
- `--mount=` syntax needs both BuildKit active *and* `# syntax=docker/dockerfile:1` at the top of the Dockerfile — missing either produces a hard parse failure.
- `hadolint` catches exactly this class of mistake alongside dozens of others (unpinned versions, shell-form `CMD`, missing `pipefail`) — run it in CI, not just locally and occasionally.
- A secret file used with `--secret src=` must never also be `COPY`'d into the build or checked into version control — keep it in `.gitignore` and `.dockerignore` both.

## Common Mistakes

- Believing a secret is safe because it's "only an env var, not a file" — the leak vector for `ENV`/`ARG` is the layer metadata itself, not a copied file.
- Forgetting the `# syntax=docker/dockerfile:1` line, then debugging a confusing "unknown flag: --mount" error that looks like a Docker installation problem but is actually a missing pragma.
- Using `--build-arg` for a genuine secret because it "felt the same" as using it for legitimate non-secret build config (a version number, a target platform) — both look identical syntactically, only one is safe to leave in history.
- Treating `hadolint` output as noise and disabling it wholesale instead of triaging and suppressing specific rule codes with a reason.
- Not pinning base image or package versions, then having a build that worked yesterday fail or silently change behavior today because `FROM python:3` or an unpinned `apt-get install` resolved to something new.
