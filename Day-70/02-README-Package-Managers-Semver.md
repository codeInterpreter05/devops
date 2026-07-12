# Day 70 — Artifact Management: Package Registries (Nexus/Artifactory) and Semantic Versioning in CI

**Phase:** 2 – CI/CD & Security | **Week:** W11 | **Domain:** CI/CD

## Brief

Not every artifact is a container image — most orgs also need somewhere to publish and consume internal Python packages, npm packages, Maven/Java artifacts, and more, without depending entirely on the public PyPI/npmjs.com registries (which are a supply-chain and availability risk if a build depends on them directly). **Nexus** and **Artifactory** are the two dominant "universal artifact repository manager" products that solve this. Layered on top of any artifact system, **semantic versioning** is the convention that makes automated version bumps and dependency resolution actually trustworthy instead of guesswork.

## Why you need a private package registry at all

Pulling directly from public PyPI/npm in every CI build and every production deploy has three real risks:
- **Availability**: if PyPI/npmjs has an outage (it happens), your builds and deploys stop too, for a dependency completely outside your control.
- **Supply chain**: a compromised or typosquatted public package (dependency confusion attacks specifically target this) can get pulled straight into your build with no intermediate check.
- **Governance**: no central point to apply license policy, vulnerability scanning, or an internal approval process before a new dependency is usable org-wide.

A private registry (Nexus/Artifactory) sits in front of these public sources as a **caching proxy** *and* hosts your own **internal packages** (private libraries shared across teams) in the same product — one tool, two roles.

## Nexus Repository / JFrog Artifactory — universal artifact managers

Both products support the same core concept: three repository *types* layered together into one address your build tools point at:

- **Hosted repository** — where your own internally-built packages live (the private PyPI packages your platform team publishes, internal npm packages, internal Maven artifacts).
- **Proxy repository** — a caching mirror of a public registry (PyPI, npmjs, Maven Central, Docker Hub). The first request for a package fetches and caches it; every subsequent request across your entire org is served from the local cache — faster, and resilient to the public registry being briefly unavailable.
- **Group repository** — a single virtual endpoint that transparently merges hosted + proxy repos, so client tools (`pip`, `npm`, `docker`) only need one URL configured, and never need to know or care whether a given package came from your team or from upstream.

```bash
# pip pointed at a Nexus/Artifactory group repo instead of pypi.org directly
pip install --index-url https://nexus.mycompany.com/repository/pypi-group/simple mypackage

# npm pointed at an Artifactory npm registry
npm config set registry https://artifactory.mycompany.com/artifactory/api/npm/npm-group/

# Publishing an internal package to the hosted repo
twine upload --repository-url https://nexus.mycompany.com/repository/pypi-internal/ dist/*
```

**Docker/OCI proxying works the same way**: configure your container runtime/CI to pull `docker.io/library/node` through `nexus.mycompany.com/docker-proxy/library/node` instead of directly from Docker Hub — this is one of the most common fixes for the Docker Hub rate-limiting problem from file 1, since the whole org shares one cached copy instead of every CI runner hitting Docker Hub independently.

## Semantic versioning (SemVer) — the contract behind automated releases

SemVer's format is `MAJOR.MINOR.PATCH` (e.g., `2.4.1`), and each position carries a **specific, binding meaning**:

- **MAJOR** — incompatible/breaking API changes. Consumers must expect to update their own code.
- **MINOR** — new functionality added in a backward-compatible way. Safe to adopt without code changes.
- **PATCH** — backward-compatible bug fixes only. Always safe to adopt.

This contract is what makes automated dependency resolution and CI version bumping *trustworthy* rather than guesswork: a consumer pinning `^2.4.0` (npm-style caret range, meaning "anything `>=2.4.0 <3.0.0`") is relying on the publisher actually honoring SemVer's meaning — if a "patch" release secretly contains a breaking change, every consumer auto-updating on that range breaks simultaneously and unexpectedly.

### Automating version bumps in CI with conventional commits

Manually deciding "is this a major, minor, or patch bump" on every release doesn't scale and is inconsistent across people. **Conventional Commits** (a commit message convention: `feat:`, `fix:`, `feat!:`/`BREAKING CHANGE:`) plus a tool like **semantic-release** automates the entire decision:

```
feat: add support for custom timeout config       -> triggers a MINOR bump
fix: correct off-by-one error in pagination        -> triggers a PATCH bump
feat!: remove deprecated v1 API endpoints          -> triggers a MAJOR bump
```

```yaml
# GitHub Actions: automated semantic-release job
jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write   # needed to create the git tag/release
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }   # full history needed to compute the version bump
      - uses: actions/setup-node@v4
      - run: npx semantic-release
```

`semantic-release` inspects every commit message since the last tag, determines the highest-severity bump type present, computes the new version number, generates a changelog, creates the Git tag, and publishes the package — all without a human manually deciding "is this 1.2.0 or 1.3.0."

## Points to Remember

- Nexus/Artifactory serve two roles at once: a caching **proxy** in front of public registries (availability + supply-chain protection) and a **host** for your own internal packages — both exposed through one **group** endpoint your tools point at.
- Proxying Docker Hub through your own registry is one of the most effective fixes for anonymous pull-rate-limit failures in CI, since the whole org shares one cached pull instead of every runner hitting Docker Hub independently.
- SemVer's MAJOR.MINOR.PATCH is a **contract**, not just a numbering convention — automated tooling and consumer version ranges (`^2.4.0`) only work correctly if publishers honor what each position is supposed to mean.
- Conventional Commits (`feat:`, `fix:`, `feat!:`) plus `semantic-release`-style tooling automates the entire version-bump decision from commit history, removing human guesswork and inconsistency from releases.
- A "patch" release that secretly contains a breaking change is a real, common incident cause — it silently breaks every consumer pinned to a permissive version range who trusted the SemVer contract.

## Common Mistakes

- Pulling directly from public PyPI/npm/Docker Hub in every CI job and production deploy, with no internal proxy — inheriting public-registry outages and dependency-confusion/typosquatting risk directly into your own build and deploy pipeline.
- Publishing a breaking change as a MINOR or PATCH release "because it was a small code change," breaking every downstream consumer who trusted the SemVer contract and pinned a permissive range like `^2.4.0`.
- Manually deciding version bumps release after release with no consistent rule, leading to version numbers that don't actually reflect the real compatibility impact of a given release — undermining the entire point of SemVer.
- Configuring a Nexus/Artifactory proxy but never verifying cache behavior — assuming a newly published upstream version is immediately available through the proxy, when some proxy configurations cache "not found" responses for a period and need an explicit cache-invalidation or a wait for TTL expiry.
- Treating conventional-commit-driven automated releases as "no review needed" — semantic-release still needs commit message discipline (and ideally commit-message linting in CI) to work correctly; sloppy commit messages produce sloppy, incorrect version bumps just as easily as manual guessing did.
