# Day 66 — Cheatsheet: Shift-Left Security

## SAST

```bash
# Semgrep
pip install semgrep
semgrep --config p/security-audit --config p/owasp-top-ten .

# Bandit (Python-specific)
pip install bandit
bandit -r ./src -ll        # -ll = medium+ severity only

# Snyk Code
snyk code test
```

```yaml
# CI (GitHub Actions)
- uses: semgrep/semgrep-action@v1
  with: { config: p/security-audit }
```

## DAST

```bash
# OWASP ZAP baseline scan against a running target
docker run -t zaproxy/zap-stable zap-baseline.py -t https://staging.example.com
```

```yaml
- uses: zaproxy/action-baseline@v0.12.0
  with:
    target: 'https://staging.example.com'
    fail_action: true
```

## Dependency scanning (SCA)

```bash
# Snyk
snyk test
snyk monitor      # continuous monitoring of a project's manifest

# npm built-in
npm audit
npm audit fix

# GitHub Dependabot - .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule: { interval: "daily" }
```

## Secret scanning

```bash
# Gitleaks
gitleaks detect --source . -v                 # scan current + history
gitleaks protect --staged                     # pre-commit style, staged files only

# TruffleHog
trufflehog git file://. --only-verified       # scan full git history, verify against live APIs
trufflehog filesystem . --only-verified

# detect-secrets
detect-secrets scan > .secrets.baseline
detect-secrets audit .secrets.baseline
```

```yaml
# CI (GitHub Actions) — needs full history!
- uses: actions/checkout@v4
  with: { fetch-depth: 0 }
- uses: gitleaks/gitleaks-action@v2
```

## Pre-commit hooks

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.4
    hooks: [{ id: gitleaks }]
  - repo: https://github.com/PyCQA/bandit
    rev: '1.7.9'
    hooks: [{ id: bandit, args: ['-ll'] }]
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.5.0
    hooks: [{ id: detect-secrets, args: ['--baseline', '.secrets.baseline'] }]
```

```bash
pip install pre-commit
pre-commit install              # wire into .git/hooks
pre-commit run --all-files      # manual full-repo run
git commit --no-verify          # bypass (know this exists — CI must still catch it)
```

## Incident response: leaked credential (do in this order)

```
1. Revoke/rotate the credential at the provider IMMEDIATELY
   aws iam delete-access-key --access-key-id <ID> --user-name <user>
2. Check usage logs for unauthorized activity since exposure
   aws cloudtrail lookup-events --lookup-attributes AttributeKey=AccessKeyId,AttributeValue=<ID>
3. Remove secret from current code + Git history
   git filter-repo / BFG Repo-Cleaner
4. Rotate any co-located/derivable credentials
5. Add/verify secret scanning is in place to prevent recurrence
```

## Git history rewrite (only after revocation)

```bash
pip install git-filter-repo
git filter-repo --replace-text replacements.txt   # replacements.txt: old-secret==>REMOVED
# or: BFG Repo-Cleaner
java -jar bfg.jar --replace-text replacements.txt
git push --force   # required after history rewrite — coordinate with all collaborators first
```
