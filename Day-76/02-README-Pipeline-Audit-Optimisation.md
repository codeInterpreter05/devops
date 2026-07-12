# Day 76 — CI/CD & DevSecOps Review: Pipeline Audit & Optimisation

**Phase:** 2 – CI/CD & Security | **Week:** W12 | **Domain:** Review

## Brief

Today's hands-on activity — auditing someone else's open-source GitHub Actions pipeline and filing 3 real security-improvement PRs — is deliberately different from every other lab in this phase, because it forces you to apply everything you've learned to code you didn't write, under real constraints (you can't ask the author "why did you do it this way," and a maintainer will actually judge whether your PR is worth merging). This is close to what a security review or a platform-team pipeline audit looks like on a real job, and it's a strong portfolio artifact — a merged security PR to a public repo is concrete, verifiable evidence of applied skill that a resume bullet point isn't.

## A checklist for auditing someone else's pipeline

Work through a real `.github/workflows/*.yml` file from an active open-source repo against this list, drawn directly from Phase 2's content:

**Supply-chain / trust boundary issues**
- [ ] Are third-party Actions pinned to a **full commit SHA** (`uses: actions/checkout@8f4b7f8...`) or a mutable tag (`uses: actions/checkout@v4`)? Tags can be moved by the action's maintainer (or an attacker who compromises their account) to point at malicious code without your workflow file changing at all.
- [ ] Does `pull_request_target` (which runs with the **base** repo's secrets and permissions, not the fork's) ever check out and execute the **fork's** untrusted code? This is one of the most common real GitHub Actions vulnerabilities — a PR from a stranger can exfiltrate your repo's secrets if `pull_request_target` checks out and runs their branch.
- [ ] Is `permissions:` scoped down (ideally per-job), or left at the (historically over-permissive) default?

**Secrets handling**
- [ ] Are secrets ever echoed to logs (`run: echo $MY_SECRET` for "debugging") or passed as CLI arguments (visible in process listings / some log output) instead of environment variables?
- [ ] Does the workflow use long-lived cloud credentials (static AWS keys in a secret) where OIDC federation (`permissions: id-token: write` + `aws-actions/configure-aws-credentials`) could eliminate the stored credential entirely?

**Pipeline gating (Day 73 content)**
- [ ] Do later stages (build, deploy) actually depend (`needs:`) on earlier gates (test, SAST, scan), or can they run even if an earlier stage failed/was skipped?
- [ ] Is there a vulnerability scan at all? If so, does it actually fail the build (`exit-code: 1`), or does it run and get ignored (`continue-on-error: true` with no follow-up)?

**Efficiency**
- [ ] Is dependency/build caching used (`actions/cache`, or built-in caching like `docker/build-push-action`'s `cache-from`/`cache-to`)? Its absence is usually the single biggest, easiest win.
- [ ] Are independent jobs actually parallelized, or unnecessarily serialized via unneeded `needs:` edges?

## Filing a real security-improvement PR

A good security PR on someone else's repo is small, well-justified, and low-risk to merge — not a wholesale rewrite:

```yaml
# Before: mutable tag, vulnerable to upstream tag-repointing
- uses: actions/checkout@v4

# After: pinned to commit SHA, with the version as a comment for readability
- uses: actions/checkout@8edcb1bdb4e267140fa742c62e395cd74f332d5 # v4.1.7
```

```yaml
# Before: pull_request_target checking out fork code directly — dangerous
on: pull_request_target
jobs:
  test:
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.sha }}   # untrusted fork code, running with base-repo secrets

# After: use pull_request (runs with the fork's own limited permissions/secrets, no access to base repo secrets)
# or, if pull_request_target is genuinely required (e.g., to comment on the PR),
# never check out/execute the fork's code within that privileged context —
# split into a separate, unprivileged workflow that only reads metadata.
on: pull_request
```

```yaml
# Before: scan runs but never blocks
- uses: aquasecurity/trivy-action@0.24.0
  with:
    exit-code: '0'          # never fails the build regardless of findings

# After: scan blocks on real findings
- uses: aquasecurity/trivy-action@0.24.0
  with:
    severity: CRITICAL,HIGH
    exit-code: '1'
```

Structure the PR description the way a maintainer wants to see it: **what's the risk, why does this specific change reduce it, and does it change any existing behavior a maintainer needs to sign off on** (e.g., pinning to SHA means the maintainer now needs to manually bump the pin on future updates — flag that trade-off explicitly rather than let them discover it).

## Pipeline optimisation: the recurring cheap wins

Independent of security, most public pipelines share the same efficiency gaps, roughly in order of impact-to-effort ratio:

1. **No caching** — re-downloading the same npm/pip/Maven dependencies or re-pulling the same Docker layers on every single run. Adding `actions/cache` keyed on the lockfile hash, or `cache-from`/`cache-to: type=gha` for Docker builds, is usually a 30-second change with a large wall-clock payoff.
2. **Unnecessary serialization** — jobs chained with `needs:` that don't actually depend on each other's output, just historically written in the order someone thought of them. Removing an unneeded `needs:` edge lets independent jobs run concurrently.
3. **Running the full test suite on every push to every branch**, including branches with no open PR — scoping `on: push` to specific branches, or `on: pull_request` only, avoids burning CI minutes on work nobody will look at.
4. **Matrix builds with unnecessary combinations** — testing against every historical language/OS version combination when only the currently supported subset matters, inflating total runtime and cost linearly with matrix size for little real benefit.

## Points to Remember

- Pinning third-party Actions to a full commit SHA (not a tag) closes a real, exploited supply-chain vector — a tag can be moved to point at different code without your workflow file changing.
- `pull_request_target` running untrusted fork code is one of the most common serious real-world GitHub Actions vulnerabilities — it grants base-repo secrets/permissions to a workflow that checks out attacker-controlled code.
- A scan step that runs but is configured with `exit-code: 0` (or `continue-on-error: true` with no downstream check) provides zero actual protection — it's compliance theater unless it can fail the build.
- A good external security PR is small, explains the specific risk and fix, and explicitly flags any maintenance trade-off the change introduces (like needing to manually bump pinned SHAs going forward).
- Caching and removing unnecessary job serialization are almost always the highest-value, lowest-risk optimizations to suggest or implement.

## Common Mistakes

- Auditing a pipeline only for what it does right and missing what it *doesn't do at all* — a missing SAST/scan step is a bigger gap than a misconfigured one, but it's easy to overlook when scanning line-by-line instead of checking against a completeness checklist.
- Submitting a huge, sweeping "security rewrite" PR to an unfamiliar open-source project instead of 2-3 small, individually justified, easy-to-review changes — maintainers are far more likely to merge small, well-explained diffs.
- Flagging `pull_request_target` usage as a blanket problem without checking whether the workflow actually checks out/executes the fork's code — `pull_request_target` alone isn't the vulnerability; combining it with checking out and running untrusted fork content is.
- Recommending SHA-pinning without mentioning the maintenance cost (manually updating pins on future version bumps, often with a bot like Dependabot/Renovate configured to do this automatically) — presenting only the upside makes the PR look naive to an experienced maintainer.
- Optimizing for speed by disabling caching invalidation checks or skipping tests on "obviously safe" branches — this trades real safety for speed in a way that isn't actually a good trade.
