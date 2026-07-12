# Day 69 — Resources: Jenkins for Legacy Environments

## Primary (assigned)

- **Jenkins documentation: Pipeline** (jenkins.io/doc/book/pipeline) — free, the assigned starting point for this day. Covers declarative and scripted syntax, the Kubernetes plugin, and shared libraries directly from the source.

## Deepen your understanding

- **Jenkins Kubernetes plugin documentation** (github.com/jenkinsci/kubernetes-plugin) — the authoritative reference for dynamic pod agent YAML, multi-container patterns, and pod templates.
- **"Extending with Shared Libraries" (Jenkins docs)** — the official guide to the `vars/`/`src/`/`resources/` structure and how library versioning/loading works.
- **CloudBees' Jenkinsfile best practices guides** — CloudBees (the primary commercial Jenkins vendor) publishes detailed guidance on structuring large-scale Jenkinsfiles and shared libraries that goes beyond the bare docs.
- **GitHub's official "Migrating from Jenkins" guide** (docs.github.com) — a direct, practical mapping guide for teams making this exact migration, worth reading right after this day's core Jenkins material.

## Reference / lookup

- **Jenkins Pipeline Syntax "Snippet Generator"** (built into any running Jenkins instance, under Pipeline Syntax) — generates correct Groovy syntax for any pipeline step interactively, invaluable when you don't remember exact step arguments.
- **Blue Ocean documentation** (jenkins.io/doc/book/blueocean) — for recognizing the UI when you encounter it in an inherited Jenkins install.

## Practice

- Stand up Jenkins via the official Helm chart on a local `kind` cluster (as in this day's lab) and wire a real multi-container Kubernetes pod agent pipeline — this is the most realistic way to build muscle memory before touching a production instance.
- Take a GitHub Actions workflow you've already written earlier in this course and reverse it into an equivalent Jenkinsfile (and vice versa) — translating in both directions solidifies the conceptual mapping far better than reading it once.
