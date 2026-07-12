# Day 9 — Lab: Git Deep Dive II — Workflows

**Goal:** Wire up a real `pre-commit` hook that runs `shellcheck` on shell scripts, enforce Conventional Commits with `commitlint` + `husky`, feel the difference between GitFlow and trunk-based branch operations by actually doing both, and produce a GPG-signed commit.

**Prerequisites:**
- Git installed, and a scratch repo to work in (do **not** do this lab inside a real project's `.git`):
  ```bash
  mkdir -p ~/labs/day9-git-workflows && cd ~/labs/day9-git-workflows
  git init
  git config user.name "Day 9 Lab"
  git config user.email "day9@example.com"
  ```
- Python 3 + pip (for `pre-commit`): `python3 --version`
- Node.js + npm (for `husky`/`commitlint`): `node --version`
- Install the tools:
  ```bash
  pip install pre-commit shellcheck-py     # shellcheck-py gives pre-commit a shellcheck binary with no OS package needed
  # OR, native shellcheck via a package manager:
  brew install shellcheck                  # macOS
  sudo apt install shellcheck              # Debian/Ubuntu
  pre-commit --version
  shellcheck --version
  ```
- GPG for the signing lab: `gpg --version` (install via `brew install gnupg` / `sudo apt install gnupg` if missing).

---

### Lab 1 — Install and wire up `pre-commit` with `shellcheck`

This is the core assigned hands-on activity: a working `pre-commit` hook that lints every `.sh` file before it can be committed.

1. Create a deliberately broken shell script:
   ```bash
   mkdir -p scripts
   cat > scripts/deploy.sh <<'EOF'
   #!/bin/bash
   TARGET_DIR=$1
   cd $TARGET_DIR
   rm -rf $TARGET_DIR/*
   echo "Deployed to $TARGET_DIR"
   EOF
   chmod +x scripts/deploy.sh
   ```
   This script has real bugs `shellcheck` will catch: unquoted `$TARGET_DIR` (word-splitting/globbing risk), no check that `cd` succeeded before the destructive `rm -rf`, and no `set -euo pipefail`.

2. Run `shellcheck` directly first, to see the raw output before any hook is involved:
   ```bash
   shellcheck scripts/deploy.sh
   ```
   You should see warnings like `SC2086: Double quote to prevent globbing and word splitting` for every unquoted `$TARGET_DIR`.

3. Create `.pre-commit-config.yaml` at the repo root:
   ```yaml
   repos:
     - repo: https://github.com/pre-commit/pre-commit-hooks
       rev: v4.6.0
       hooks:
         - id: trailing-whitespace
         - id: end-of-file-fixer
         - id: check-added-large-files

     - repo: https://github.com/shellcheck-py/shellcheck-py
       rev: v0.10.0.1
       hooks:
         - id: shellcheck
           args: ["--severity=warning"]
   ```

4. Install the hook and run it against the whole repo once (new hooks should always be run against everything first, since `pre-commit` normally only checks staged files):
   ```bash
   pre-commit install
   pre-commit run --all-files
   ```
   The `shellcheck` hook should **fail** and print the same `SC2086` warnings, blocking would-be commits.

5. Stage and try to commit the broken script — confirm the commit is actually blocked:
   ```bash
   git add .pre-commit-config.yaml scripts/deploy.sh
   git commit -m "add deploy script"
   ```
   The commit should abort with the `shellcheck` hook reporting failures.

6. Fix the script properly and re-attempt:
   ```bash
   cat > scripts/deploy.sh <<'EOF'
   #!/bin/bash
   set -euo pipefail

   TARGET_DIR="$1"

   if [ -z "$TARGET_DIR" ]; then
     echo "Usage: $0 <target-dir>" >&2
     exit 1
   fi

   cd "$TARGET_DIR"
   rm -rf -- "${TARGET_DIR:?}"/*
   echo "Deployed to $TARGET_DIR"
   EOF
   git add scripts/deploy.sh
   git commit -m "feat(deploy): add deploy script with shellcheck-clean quoting"
   ```

**Success criteria:** `pre-commit run --all-files` passes cleanly, the first commit attempt with the broken script was actually blocked (not just warned), and the fixed script commits successfully.

---

### Lab 2 — Enforce Conventional Commits with `commitlint` + `husky`

1. Initialize a `package.json` and install the tooling:
   ```bash
   npm init -y
   npm install --save-dev husky @commitlint/cli @commitlint/config-conventional
   ```

2. Create `commitlint.config.js`:
   ```js
   module.exports = {
     extends: ['@commitlint/config-conventional'],
     rules: {
       'header-max-length': [2, 'always', 72],
       'subject-case': [2, 'never', ['upper-case']],
     },
   };
   ```

3. Wire up `husky` and the `commit-msg` hook:
   ```bash
   npx husky init
   echo 'npx --no-install commitlint --edit "$1"' > .husky/commit-msg
   chmod +x .husky/commit-msg
   git add package.json package-lock.json commitlint.config.js .husky
   git commit -m "chore(hooks): wire up commitlint via husky"
   ```
   (This commit message must itself already be conventional — if it's rejected, fix the message rather than bypassing the hook.)

4. Prove enforcement works — try an intentionally invalid message:
   ```bash
   git commit --allow-empty -m "made some changes"
   ```
   `commitlint` should reject it (`subject may not be empty` / `type must be one of [...]` depending on the exact violation) and no commit should be created.

5. Now use a valid Conventional Commit message:
   ```bash
   git commit --allow-empty -m "fix(deploy): correct quoting bug caught by shellcheck"
   ```
   This should succeed.

6. Test a `BREAKING CHANGE` footer and confirm `commitlint` accepts the multi-line form:
   ```bash
   git commit --allow-empty -m "feat(api)!: change deploy() signature

   BREAKING CHANGE: deploy() now requires an explicit target directory argument"
   ```

**Success criteria:** a non-conventional message is rejected before a commit object is created, and both a plain `feat:`/`fix:` message and a `BREAKING CHANGE` footer are accepted. You can explain out loud what `semantic-release` would do differently for each of the three commits you just made (patch bump, patch bump, major bump).

---

### Lab 3 — Simulate GitFlow vs. trunk-based branch operations

Do both models against the same starting point so the contrast is concrete, not theoretical.

**3a — GitFlow-style flow:**
```bash
git checkout -b develop
git checkout -b feature/currency-support develop
echo "// multi-currency logic" >> scripts/deploy.sh
git commit -am "feat(payments): start multi-currency support"
git checkout develop
git merge --no-ff feature/currency-support
git checkout -b release/1.1.0 develop
echo "1.1.0" > VERSION
git commit -am "chore(release): bump version to 1.1.0"
git checkout main 2>/dev/null || git checkout -b main
git merge --no-ff release/1.1.0
git checkout develop
git merge --no-ff release/1.1.0
```
Notice how many branches exist and how many merge points were required to get one feature from idea to `main`.

**3b — Trunk-based style, same feature:**
```bash
git checkout main
git checkout -b add-currency-flag
echo "if (featureFlags.multiCurrency) { /* new logic */ }" >> scripts/deploy.sh
git commit -am "feat(payments): add multi-currency behind feature flag"
git checkout main
git merge --ff-only add-currency-flag   # fails if main moved on; that's the point — keep branches short-lived
git branch -d add-currency-flag
```
Note the `--ff-only`: it succeeds cleanly here because the branch is only one commit old and `main` hasn't moved — that's what "short-lived" buys you. Try recreating the branch, letting a second commit land on `main` in between, and confirm `--ff-only` now refuses (demonstrating why long-lived branches make this harder).

**Success criteria:** you can explain, using the branches you just created, why GitFlow needed both a `develop` merge and a `main` merge for one feature, while the trunk-based version needed one branch and one fast-forward — and why the trunk-based version *requires* the feature flag comment/gate to be safe to ship (the code path exists on `main` immediately, active or not).

---

### Lab 4 — Branch protection concepts and a GPG-signed commit

**4a — Branch protection (concept walkthrough, since this lab repo is local-only):**

On a real GitHub repo (use any personal scratch repo you can push this lab to, or read along if you don't have one handy), go to **Settings → Branches → Add branch protection rule** for `main` and enable, in this order, explaining out loud why each one matters before you check it:
1. Require a pull request before merging, with at least 1 approval.
2. Require status checks to pass — pick a check name (e.g., `pre-commit`) and mark it required.
3. Require branches to be up to date before merging.
4. Do not allow force pushes.
5. Do not allow deletions.

**4b — GPG-signed commit, for real:**
```bash
gpg --full-generate-key         # choose RSA and RSA, 4096 bits, no expiration for lab purposes
gpg --list-secret-keys --keyid-format=long
```
Copy the key ID (the hex string after `rsa4096/`), then:
```bash
git config user.signingkey <YOUR_KEY_ID>
git config commit.gpgsign true
git commit --allow-empty -m "chore: verify GPG signing works"
git log --show-signature -1
```

**Success criteria:** `git log --show-signature -1` prints `gpg: Good signature from ...` for your test commit, and you can state in one sentence what "Verified" on GitHub actually proves (the commit was produced by the holder of a specific registered private key) versus what it does not prove (that the code in the commit is safe or correct).

---

### Cleanup

```bash
cd ~/labs
rm -rf day9-git-workflows
# optional: remove the lab GPG key if you don't want to keep it around
gpg --list-secret-keys --keyid-format=long   # find the key ID again
gpg --delete-secret-and-public-key <YOUR_KEY_ID>
```

### Stretch challenge

Add a `pre-push` hook (via `husky`, `.husky/pre-push`) that refuses to push if any commit being pushed has a message not matching Conventional Commits, *even if* someone bypassed `commit-msg` locally with `--no-verify` — then explain in one sentence why this still isn't equivalent to a server-side `pre-receive`/branch-protection check, and what class of bypass it still can't stop (hint: what if the whole `.husky` directory itself is deleted or never installed on the pushing machine?).
