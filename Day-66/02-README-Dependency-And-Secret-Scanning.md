# Day 66 — Shift-Left Security: Dependency Scanning (SCA) & Secret Scanning

**Phase:** 2 – CI/CD & Security | **Week:** W10 | **Domain:** DevSecOps | **Flag:** ⚡ Interview-critical

## Brief

Most of the code running in any real application isn't code your team wrote — it's third-party dependencies, often dozens or hundreds of transitive packages deep. And the single most common real-world security incident triggering today's actual interview question ("a developer committed an AWS key to a public repo") isn't a sophisticated exploit — it's a plain credential sitting in plaintext in Git history. Dependency scanning (SCA) and secret scanning are the two shift-left controls that address these two extremely common, high-frequency risk categories directly.

## Dependency scanning / Software Composition Analysis (SCA)

**SCA** identifies every open-source/third-party component your application actually uses (direct *and* transitive dependencies), and checks each one against databases of known vulnerabilities (CVEs) and license obligations.

```yaml
# GitHub Actions example: Snyk
jobs:
  sca:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Snyk to check for vulnerabilities
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high
```

```bash
# GitHub's built-in Dependabot (no separate tool needed) - .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule: { interval: "daily" }
```

**How it actually works**: the scanner builds a full dependency tree (parsing lockfiles like `package-lock.json`, `poetry.lock`, `go.sum`) — not just your direct `package.json` entries, since a vulnerability three levels deep in a transitive dependency is just as exploitable as one in a package you imported directly. Each resolved package+version is checked against vulnerability databases (the National Vulnerability Database, GitHub Advisory Database, Snyk's own curated database), and matches are reported with a severity score (typically CVSS-based) and, critically, **whether a fixed version is available** — the actual actionable remediation.

**Why transitive dependencies matter so much in practice**: the infamous **Log4Shell** (CVE-2021-44228) vulnerability lived in `log4j-core`, a logging library that countless applications depended on *transitively* — many affected teams didn't even know they had Log4j in their dependency tree at all until SCA tooling (or the ensuing scramble) told them, because it was three or four levels removed from anything they'd directly declared. This is the canonical real-world argument for why SCA has to walk the *full* resolved dependency graph, not just top-level manifests.

**License scanning** is SCA's other job, often overlooked: flagging dependencies under licenses incompatible with your product's licensing model (e.g., a copyleft GPL-licensed library pulled into proprietary closed-source software) — a legal/compliance risk, not a security vulnerability, but bundled into the same tooling because it requires the same dependency-tree analysis.

## Secret scanning (Gitleaks, TruffleHog)

Secret scanning looks for credentials — API keys, private keys, database connection strings, cloud provider access keys — accidentally committed into a Git repository, including **historical commits**, not just the current working tree.

```yaml
# GitHub Actions example: Gitleaks
jobs:
  secret-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0   # full history — required to scan past commits, not just HEAD
      - uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

```bash
# TruffleHog, scanning full git history locally
trufflehog git file://. --only-verified
```

**How it actually works**: these tools combine **regex pattern matching** against known credential formats (AWS keys start with `AKIA`, GitHub tokens have recognizable prefixes like `ghp_`, private keys have a distinctive PEM header) with **entropy analysis** (a random-looking, high-entropy string is more likely to be a real secret than a low-entropy string like `password123`, catching credential formats without a known fixed pattern). TruffleHog's `--only-verified` flag goes a step further: it doesn't just pattern-match, it actually **attempts to use the discovered credential against the real provider's API** (e.g., a live AWS API call) to confirm whether it's still active — turning "this looks like a secret" into "this secret is confirmed live and exploitable right now," which drastically cuts false positives and correctly prioritizes urgency.

**Why scanning Git history (not just the current diff) is essential**: `git rm` or even a follow-up commit that removes a secret from the *current* file does **not** remove it from history — anyone who clones the repo, or runs `git log -p`, can still recover it from an old commit. A secret scanner that only checks the latest diff on each PR will miss a secret that was committed and then "fixed" three commits later — the secret is still sitting there, permanently, unless the actual Git history is rewritten (`git filter-repo`/BFG Repo-Cleaner) and force-pushed, and even then, anyone who already has a local clone still has it.

### The actual incident-response reality (ties directly to today's interview question)

The moment a credential is confirmed committed to a public (or even private, since insiders/compromised accounts still count) repo, **the only fully correct remediation is rotating/revoking the credential at the provider** — rewriting Git history to remove it is necessary hygiene but is **not sufficient on its own**, because:
- The credential may already have been scraped by automated bots that continuously crawl GitHub for exposed secrets (this happens within *minutes* for public repos, not hours).
- Anyone who already cloned the repo, or who saw it via a cached view/fork/CI log, retains a copy regardless of what you do to the canonical history afterward.

So the actual 5-minute response to "a developer committed an AWS key to a public repo" is, in priority order: **(1) revoke/rotate the credential at AWS immediately** (this is the only step that actually closes the exposure — everything else is cleanup), (2) check CloudTrail/access logs for any usage of that key since it was exposed, (3) remove the secret from the current code and Git history, (4) rotate any other credentials that might have been co-located or derivable, (5) add secret scanning to prevent recurrence if it wasn't already in place. Rewriting history *before* revoking the key is a common instinctive-but-wrong first move — it fixes the repo but leaves the actual live credential valid and exploitable the entire time you spend on Git surgery.

## Points to Remember

- SCA scans the *full resolved dependency tree* (including transitive dependencies), not just direct manifest entries — Log4Shell is the canonical example of why this matters.
- SCA findings should be actioned based on whether a fixed version exists and the severity/exploitability, not just raw CVE count — not every CVE in a dependency you use is actually reachable/exploitable in your specific usage.
- Secret scanners combine regex pattern matching (known credential formats) with entropy analysis (catches unknown-format high-randomness strings) — and tools like TruffleHog can further verify a secret is still live against the real provider API.
- Removing a secret from the current file/diff does **not** remove it from Git history — scanning (and cleanup, if needed) must cover full history, not just the latest commit.
- The correct first response to an exposed credential is always **revoke/rotate at the provider immediately** — Git history cleanup is necessary hygiene but happens after, never instead of, revocation.

## Common Mistakes

- Running SCA against only direct dependencies (top-level manifest) and missing vulnerabilities several levels deep in transitive dependencies — exactly the Log4Shell pattern.
- Treating every SCA-flagged CVE as equally urgent regardless of whether the vulnerable code path is actually reachable in your usage — leads to alert fatigue and ignored dashboards.
- Believing `git rm secret.txt && git commit` actually removes a secret from the repository — it only removes it from the current tree; the blob is still recoverable from history until it's rewritten and force-pushed (and even then, existing clones retain it).
- Rewriting Git history to scrub a leaked secret *before* rotating the actual credential at the provider — the leaked key remains fully valid and exploitable the entire time, regardless of what the repo looks like.
- Only scanning for secrets on new commits/PRs going forward, without ever running a full-history scan on the existing repository — leaving years of already-committed secrets undiscovered.
- Ignoring low-confidence/unverified secret-scanner findings entirely instead of triaging them — some real secrets don't match a known provider format and only surface via entropy heuristics, which have a higher false-positive rate but still need a human look, not an automatic dismiss.
