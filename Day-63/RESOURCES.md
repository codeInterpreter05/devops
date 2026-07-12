# Day 63 — Resources: GitLab CI Deep Dive

## Primary (assigned)

- **GitLab CI/CD documentation** (docs.gitlab.com/ee/ci/) — free, the assigned starting point. Start with "CI/CD YAML syntax reference" and "Pipeline architecture" for the `stages`/`rules`/`needs` model covered today.

## Deepen your understanding

- **GitLab "Optimize GitLab CI/CD pipeline configuration"** (docs.gitlab.com/ee/ci/pipelines/pipeline_efficiency.html) — covers `needs:`-based DAG pipelines, caching, and parallelization patterns beyond the basics.
- **GitLab "`rules` reference"** (docs.gitlab.com/ee/ci/yaml/#rules) — the exhaustive, authoritative spec on rule evaluation order, `changes:`, and every `when:` value.
- **GitLab Container Registry documentation** (docs.gitlab.com/ee/user/packages/container_registry/) — covers Docker-in-Docker setup, `kaniko`/`buildah` alternatives, and cleanup policies in depth.
- **"GitLab CI/CD examples" repo** (docs.gitlab.com/ee/ci/examples/) — GitLab's own collection of real-world `.gitlab-ci.yml` templates across languages/frameworks, good for pattern-matching your own pipeline against a known-good example.

## Reference / lookup

- **Predefined CI/CD variables reference** (docs.gitlab.com/ee/ci/variables/predefined_variables.html) — the full list of `$CI_*` variables available in every job.
- **GitLab Runner executors documentation** (docs.gitlab.com/runner/executors/) — details on `docker`, `shell`, `kubernetes`, and `docker-autoscaler` executor tradeoffs.

## Practice

- Take any existing GitHub Actions workflow (your own from Day 61, or any public repo's) and migrate it to `.gitlab-ci.yml` from scratch without looking at a reference translation table first — then check your result against the mapping table in file 3 of today's notes.
- GitLab's own public repositories (gitlab.com/gitlab-org) have large, real `.gitlab-ci.yml` files you can read to see `extends`/`include:`/`rules:` used at real production scale, well beyond a lab-sized example.
