# Day 76 — Cheatsheet: CI/CD & DevSecOps Review

## Gaps-review method (fast recall check)

```
1. Closed-book dump (5 min/domain, no notes)  -> compare vs. real notes -> circle gaps
2. Explain out loud, 90 sec, unscripted, recorded -> listen for hedging phrases
3. Cold-answer past "Interview Question to prep" lines, timed <2 min each
4. "Can I build it or only describe it?" test, per tool
```

## Phase 2 domain checklist (use for the closed-book dump)

```
[ ] CI pipeline gating design (needs:, fail-fast vs informational stages)
[ ] SAST / SCA (Semgrep, CodeQL, dependency scanning)
[ ] Container image scanning (Trivy, severity thresholds, exit-code gating)
[ ] Secrets management in CI (OIDC vs static creds, secret scanning)
[ ] Artifact signing / provenance (cosign, SLSA, digest vs tag)
[ ] GitOps (ArgoCD sync policies, selfHeal, drift)
[ ] Compliance-as-code (CIS, kube-bench, Checkov, Prowler, Security Hub)
[ ] SOC2 / GDPR technical controls
[ ] DB migrations (expand-contract, Flyway/Liquibase/Alembic, K8s Jobs)
```

## Pipeline security audit checklist

```
Supply chain:
  [ ] third-party Actions pinned to commit SHA, not mutable tag
  [ ] pull_request_target never checks out + executes fork code directly
  [ ] permissions: scoped per-job, not workflow-wide write-all

Secrets:
  [ ] no secrets echoed to logs or passed as bare CLI args
  [ ] OIDC federation used instead of long-lived static cloud creds where possible

Gating:
  [ ] deploy/build actually needs: earlier test/SAST/scan jobs
  [ ] scan step's exit-code actually fails the build on findings (not exit-code: '0')

Efficiency:
  [ ] dependency/build caching present (actions/cache, docker cache-from/to)
  [ ] no unnecessary needs: serialization between independent jobs
  [ ] CI doesn't run full suite on every branch push with no PR
  [ ] matrix builds scoped to actually-supported versions only
```

## Grep-style checks to find issues fast

```bash
grep -rn "uses:.*@v[0-9]" .github/workflows/     # mutable tag pins (should be SHA)
grep -rn "pull_request_target" .github/workflows/
grep -rn "exit-code: '0'" .github/workflows/
grep -rn "continue-on-error: true" .github/workflows/
grep -rn "permissions: write-all" .github/workflows/
```

## Security PR — before/after pattern

```yaml
# Before
- uses: actions/checkout@v4

# After (pin to SHA, keep version as a readable comment)
- uses: actions/checkout@8edcb1bdb4e267140fa742c62e395cd74f332d5 # v4.1.7
```

## PR description template for a security fix

```
**Risk:** <what could go wrong today>
**Fix:** <the specific, minimal change>
**Trade-off:** <any maintenance cost this introduces, e.g. manual SHA bumps>
```
