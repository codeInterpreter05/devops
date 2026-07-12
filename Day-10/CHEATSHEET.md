# Day 10 — Cheatsheet: Docker Deep Dive I

## Multi-stage build syntax

```dockerfile
# syntax=docker/dockerfile:1

FROM golang:1.22 AS builder          # named stage
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o server .

FROM scratch                         # empty base, only the final stage ships
COPY --from=builder /app/server /server
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/ca-certificates.crt
ENTRYPOINT ["/server"]
```

```bash
docker build --target builder -t myapp:debug .    # build+tag only an intermediate stage
docker build --from=0 ...                         # reference an unnamed stage by index
```

## Distroless / scratch base images

```
gcr.io/distroless/static-debian12       # statically linked binaries (Go CGO_ENABLED=0, Rust musl)
gcr.io/distroless/base-debian12         # glibc + libssl, no language runtime
gcr.io/distroless/python3-debian12      # Python runtime, ENTRYPOINT ["python3"]
gcr.io/distroless/nodejs20-debian12     # Node runtime
gcr.io/distroless/java17-debian12       # JVM
gcr.io/distroless/<variant>:nonroot     # runs as unprivileged UID
gcr.io/distroless/<variant>:debug       # adds a minimal busybox shell for troubleshooting
scratch                                 # truly empty — zero bytes, no libc, no shell
```

## Python multi-stage distroless example

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

## BuildKit secrets

```bash
DOCKER_BUILDKIT=1 docker build --secret id=api_key,src=./secret.txt -t myapp .   # enable + pass secret
```

```dockerfile
# syntax=docker/dockerfile:1              # REQUIRED for --mount syntax to parse
RUN --mount=type=secret,id=api_key \
    API_KEY=$(cat /run/secrets/api_key) && \
    curl -H "Authorization: Bearer $API_KEY" https://example.com

RUN --mount=type=ssh git clone git@github.com:org/private-repo.git   # SSH agent forwarding, same pattern
```

```
NEVER do this instead (leaks into `docker history` / `docker inspect` permanently):
ARG API_KEY
ENV API_KEY=$API_KEY
```

## `.dockerignore`

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

logs/*
!logs/.gitkeep          # negation: re-include something a broader pattern excluded
```

## Instruction ordering for cache efficiency

```dockerfile
# GOOD — dependency manifest copied and installed before volatile source code
COPY package.json package-lock.json ./
RUN npm install
COPY . .

# BAD — any source change invalidates the install layer too
COPY . .
RUN npm install
```

```dockerfile
# apt-get update + install MUST be one RUN (avoids stale cached index)
RUN apt-get update && apt-get install -y --no-install-recommends curl=7.88.1-10 \
    && rm -rf /var/lib/apt/lists/*
```

## `dive` — layer inspection

```bash
dive myapp:latest              # inspect an already-built image
dive build -t myapp:latest .   # build then inspect
CI=true dive myapp:latest      # non-interactive pass/fail against thresholds

# .dive-ci config file:
# rules:
#   lowestEfficiency: 0.95
#   highestWastedBytes: 20MB
#   highestUserWastedPercent: 0.10
```

Keys in the TUI: `Tab` switch panes, arrows navigate, `Ctrl+A` toggle all-files view, `Ctrl+L` toggle layer-changes-only view.

## `hadolint` — Dockerfile linting

```bash
hadolint Dockerfile                          # local install
docker run --rm -i hadolint/hadolint < Dockerfile   # no install needed
hadolint --ignore DL3008 Dockerfile          # suppress a specific rule
```

```
DL3006   pin FROM image tag, never `latest`
DL3008   pin apt-get install package versions
DL3009   rm -rf /var/lib/apt/lists/* in the same RUN as install
DL3013   pin pip package versions
DL3042   use pip install --no-cache-dir
DL4006   set SHELL with -o pipefail when a RUN uses a pipe
DL3025   use JSON/exec form for CMD/ENTRYPOINT, not shell form
```

## Image size inspection

```bash
docker images                            # size per tag
docker images day10-naive                # filter by repo name
docker history myimage                   # per-layer size + truncated command
docker history --no-trunc myimage        # full command text per layer — check for leaked secrets here
docker inspect myimage                   # full config, including .Config.Env (ARG/ENV values persist here)
docker system df -v                      # disk usage breakdown: images, containers, volumes, build cache
docker system prune -f                   # remove dangling images/containers/cache
```
