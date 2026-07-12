# Day 66 — Lab: Shift-Left Security

**Goal:** Add secret scanning and SAST to a CI pipeline, discover and fix intentional vulnerabilities in a test repo, and practice the actual incident response for a leaked credential.

**Prerequisites:**
- A throwaway test repo (never use a real production repo for this lab — you're about to deliberately commit fake secrets and vulnerable code).
- `pip install pre-commit detect-secrets bandit`, `brew install gitleaks trufflehog semgrep` (or your OS package manager equivalents).
- Docker installed, if you want to run OWASP ZAP for a bonus DAST pass.

---

### Lab 1 — Plant intentional vulnerabilities (safely, in a throwaway repo)

1. Create a small Python or Node app with a handful of deliberately bad patterns:
   ```python
   # app.py - INTENTIONALLY VULNERABLE, lab use only, never deploy this
   import subprocess

   AWS_ACCESS_KEY_ID = "AKIAIOSFODNN7EXAMPLE"   # fake AWS-format key, for lab detection only

   def run_backup(filename):
       subprocess.run(f"tar -czf backup.tar.gz {filename}", shell=True)   # command injection

   def get_user(conn, user_id):
       query = f"SELECT * FROM users WHERE id = {user_id}"   # SQL injection
       return conn.execute(query)
   ```
2. Commit it to your throwaway repo, including in Git history (this is deliberate — you'll practice detecting it in history in Lab 3).

**Success criteria:** A repo with at least one hardcoded fake secret, one command injection pattern, and one SQL injection pattern, ready for tooling to find.

---

### Lab 2 — The core hands-on activity: add secret scanning + SAST to CI

This is today's assigned hands-on activity.

1. Add a GitHub Actions workflow:
   ```yaml
   name: Security
   on: [push, pull_request]
   jobs:
     secret-scan:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v4
           with: { fetch-depth: 0 }
         - uses: gitleaks/gitleaks-action@v2

     sast:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v4
         - name: Bandit
           run: |
             pip install bandit
             bandit -r . -ll
         - name: Semgrep
           uses: semgrep/semgrep-action@v1
           with:
             config: p/security-audit
   ```
2. Push and confirm both jobs run and **fail** on the intentional vulnerabilities from Lab 1 — read the actual findings output (Gitleaks should flag the fake AWS key; Bandit/Semgrep should flag `shell=True` and the f-string SQL query).
3. Fix each finding properly (not by suppressing):
   ```python
   import shlex, subprocess

   def run_backup(filename):
       subprocess.run(["tar", "-czf", "backup.tar.gz", filename])   # no shell=True, args as a list

   def get_user(conn, user_id):
       query = "SELECT * FROM users WHERE id = ?"   # parameterized query
       return conn.execute(query, (user_id,))
   ```
4. Remove the hardcoded key from the current file and push a fix commit. Re-run the pipeline and confirm SAST now passes — but check whether Gitleaks still flags the secret (it should, since it's still in Git history from Lab 1's commit).

**Success criteria:** A CI pipeline that fails on the original vulnerable commit and passes (SAST-wise) on the fixed commit, with you having personally read and understood each finding rather than blindly suppressing it.

---

### Lab 3 — Full Git history secret scanning and cleanup

1. Confirm the "fixed" commit from Lab 2 still leaves the fake key recoverable in history:
   ```bash
   git log -p --all | grep -i AKIA
   trufflehog git file://. --only-verified
   ```
2. Practice history rewriting to actually remove it (safe here because it's a throwaway repo with no other collaborators):
   ```bash
   pip install git-filter-repo
   git filter-repo --invert-paths --path-glob '*.py' --force   # illustrative; in practice use --replace-text for targeted secret removal
   ```
   Or use BFG Repo-Cleaner with a `replacements.txt` file mapping the fake key to `REMOVED`.
3. Re-run `trufflehog`/`gitleaks` against the rewritten history and confirm the secret is gone.

**Success criteria:** You've personally demonstrated that deleting a secret from the current file does NOT remove it from history, and that actual history rewriting is a separate, deliberate, disruptive operation (which is why revocation, not history rewriting, is always the first real response).

---

### Lab 4 — Practice the 5-minute incident response

Simulate the exact scenario in today's interview question.

1. Write out, as if for a real incident, the exact ordered steps you'd take in the first 5 minutes after discovering a real AWS key committed to a public repo (don't just recall the notes — write your own version).
2. If you have a personal AWS sandbox account, actually practice step 1 for real: create a throwaway IAM user/access key, then immediately practice revoking it:
   ```bash
   aws iam delete-access-key --access-key-id <FAKE_OR_REAL_TEST_KEY> --user-name <lab-user>
   ```
3. Practice checking for unauthorized usage (even against a key you know wasn't actually exposed, to learn the command):
   ```bash
   aws cloudtrail lookup-events --lookup-attributes AttributeKey=AccessKeyId,AttributeValue=<KEY_ID>
   ```

**Success criteria:** You have a written, ordered incident-response checklist you could execute from memory under pressure, and you've run the actual `aws iam delete-access-key` command at least once.

---

### Lab 5 — Set up pre-commit hooks

1. Add `.pre-commit-config.yaml` with `gitleaks` and `bandit` hooks (see file 3's example), then:
   ```bash
   pre-commit install
   ```
2. Try to commit a new fake secret and confirm the commit is **blocked locally** before it ever reaches `git push`.
3. Bypass it deliberately to see the escape hatch exists: `git commit --no-verify`, then confirm CI (Lab 2's workflow) still catches it — demonstrating why CI-level scanning must exist regardless of local hooks.

**Success criteria:** You've seen a pre-commit hook block a bad commit locally, and separately confirmed that bypassing it still gets caught by CI.

---

### Cleanup

```bash
# Delete the lab IAM user/keys if you created real ones
aws iam delete-access-key --access-key-id <KEY_ID> --user-name <lab-user>
aws iam delete-user --user-name <lab-user>

# Delete the throwaway repo entirely (do not reuse it — it has fake secrets baked into its history)
```

### Stretch challenge

Add an OWASP ZAP baseline DAST scan (`zaproxy/action-baseline`) against a locally running instance of your test app, and compare its findings to what SAST caught — identify at least one class of issue DAST could plausibly find that SAST structurally cannot (e.g., a misconfigured response header).
