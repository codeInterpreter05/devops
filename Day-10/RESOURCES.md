# Day 10 — Resources: Docker Deep Dive I

## Primary (assigned)

- **Docker docs: Multi-stage builds** (docs.docker.com/build/building/multi-stage/) — the assigned starting point for this day. Official reference for `FROM ... AS`, `COPY --from=`, and `--target`, straight from the source.

## Deepen your understanding

- **Docker docs: BuildKit** (docs.docker.com/build/buildkit/) — covers `--mount=type=secret`, `--mount=type=ssh`, `--mount=type=cache`, and how the BuildKit frontend/syntax versioning (`# syntax=docker/dockerfile:1`) works.
- **GoogleContainerTools/distroless** (github.com/GoogleContainerTools/distroless) — the source project for `gcr.io/distroless/*`. The README explains exactly what each variant (`static`, `base`, `cc`, language-specific images, `:nonroot`, `:debug`) does and doesn't include.
- **Docker docs: Dockerfile best practices** (docs.docker.com/build/building/best-practices/) — the official guidance on layer ordering, `.dockerignore`, and minimizing image size, complementary to today's notes.

## Reference

- **wagoodman/dive** (github.com/wagoodman/dive) — source, install instructions for every platform, and the `.dive-ci` config format for wiring efficiency thresholds into CI.
- **hadolint/hadolint** (github.com/hadolint/hadolint) — full rule list (`DL####` codes) with explanations; check here whenever a hadolint finding is unclear.
- **Docker docs: `.dockerignore` file** (docs.docker.com/build/building/context/#dockerignore-files) — exact syntax reference, including negation patterns.

## Practice

- **Refactor a real project you own.** The single highest-signal practice available: take any personal or work-adjacent repo with a Dockerfile, run `dive` and `hadolint` against it as a baseline, then apply today's techniques and remeasure. Real dependency trees surface edge cases (native extensions needing a compiler, packages without prebuilt wheels) that a toy Flask app won't.
- **hadolint's own test Dockerfiles** (in the hadolint repo's test fixtures) — a good way to see many rule violations concentrated in small, deliberately bad examples.
