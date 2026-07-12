# Day 9 — Cheatsheet: Git Deep Dive II — Workflows

## `pre-commit` setup and usage

```bash
pip install pre-commit               # install the framework
pre-commit install                    # writes .git/hooks/pre-commit -> delegates to framework
pre-commit install --hook-type commit-msg   # also wire the commit-msg stage

pre-commit run                        # run configured hooks against staged files
pre-commit run --all-files            # run against every file in the repo (always do this after adding a hook)
pre-commit run <hook-id> --all-files  # run one specific hook against everything
pre-commit autoupdate                 # bump every hook's `rev:` to its latest tag
pre-commit uninstall                  # remove the installed git hook
```

`.pre-commit-config.yaml`:
```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files

  - repo: https://github.com/shellcheck-py/shellcheck-py
    rev: v0.10.0.1
    hooks:
      - id: shellcheck
        args: ["--severity=warning"]

  - repo: https://github.com/compilerla/conventional-pre-commit
    rev: v3.4.0
    hooks:
      - id: conventional-pre-commit
        stages: [commit-msg]
```

## Conventional Commits format

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

```
feat(auth): add refresh-token rotation
fix(api): handle null pointer on empty payload
chore(deps): bump express to 4.19.2
docs(readme): clarify local setup steps
refactor(cache): extract TTL logic into helper
test(payments): add currency conversion edge cases
perf(query): add index on users.email
ci(actions): cache node_modules between runs

feat(payments)!: require currency argument

BREAKING CHANGE: charge() now requires a currency argument
```

Common `type`s: `feat`, `fix`, `chore`, `docs`, `refactor`, `test`, `perf`, `ci`, `build`, `style`.
`!` right after type/scope, or a `BREAKING CHANGE:` footer, = major version bump under SemVer automation.

| Commit type | SemVer bump (via semantic-release) |
|---|---|
| `fix` | patch |
| `feat` | minor |
| `BREAKING CHANGE` footer or `!` | major |
| `chore`/`docs`/`style`/`test`/`ci` | none by default |

## `husky` + `commitlint` setup

```bash
npm install --save-dev husky @commitlint/cli @commitlint/config-conventional
npx husky init                                        # creates .husky/, adds "prepare": "husky" to package.json

echo 'npx --no-install commitlint --edit "$1"' > .husky/commit-msg
chmod +x .husky/commit-msg
```

`commitlint.config.js`:
```js
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'header-max-length': [2, 'always', 72],
    'body-max-line-length': [2, 'always', 100],
    'scope-case': [2, 'always', 'lower-case'],
  },
};
```

```bash
echo "bad message" | npx commitlint          # lint a message manually, without committing
npx commitlint --edit .git/COMMIT_EDITMSG     # lint the in-progress commit message file directly
git commit --no-verify -m "..."               # BYPASSES all client-side hooks — for debugging only, never habit
```

## GPG signing

```bash
gpg --full-generate-key                            # generate a key pair (RSA 4096 or ed25519)
gpg --list-secret-keys --keyid-format=long          # find your key ID

git config --global user.signingkey <KEY_ID>
git config --global commit.gpgsign true             # sign every commit automatically
git config --global tag.gpgsign true                # same, for annotated tags

git commit -S -m "message"                          # sign one commit explicitly (if gpgsign isn't global)
git log --show-signature -1                         # verify locally against your GPG keyring
git tag -s v1.0.0 -m "release 1.0.0"                # signed annotated tag
git verify-commit <sha>                             # verify a specific commit's signature
git verify-tag v1.0.0                                # verify a specific tag's signature
```

## SSH-based commit signing (Git 2.34+)

```bash
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub
git config --global commit.gpgsign true

echo 'you@example.com ssh-ed25519 AAAA...' >> ~/.ssh/allowed_signers
git config --global gpg.ssh.allowedSignersFile ~/.ssh/allowed_signers
```

## Branch protection (GitHub CLI reference)

```bash
gh api repos/:owner/:repo/branches/main/protection --method PUT --input protection.json
# protection.json roughly:
# {
#   "required_status_checks": {"strict": true, "contexts": ["pre-commit", "ci"]},
#   "enforce_admins": true,
#   "required_pull_request_reviews": {"required_approving_review_count": 1, "dismiss_stale_reviews": true},
#   "restrictions": null,
#   "required_linear_history": true,
#   "allow_force_pushes": false,
#   "allow_deletions": false,
#   "required_signatures": true
# }

gh api repos/:owner/:repo/branches/main/protection/required_signatures --method POST   # require signed commits
```

## GitFlow vs. trunk-based, quick reference

```
GitFlow:     main <- release/* <- develop <- feature/*   (+ hotfix/* off main)
Trunk-based: main <- short-lived branch (hours), feature flags gate exposure
```

```bash
# GitFlow-ish
git checkout -b feature/x develop
git checkout develop && git merge --no-ff feature/x
git checkout -b release/1.1.0 develop
git checkout main && git merge --no-ff release/1.1.0
git checkout develop && git merge --no-ff release/1.1.0

# Trunk-based
git checkout -b quick-fix main
git commit -am "fix: ..."
git checkout main && git merge --ff-only quick-fix && git branch -d quick-fix
```
