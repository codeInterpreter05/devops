# Day 21 — Cheatsheet: Portfolio & Documentation

## Repo init

```bash
mkdir devops-fundamentals && cd devops-fundamentals
git init
git branch -M main
git config user.name "Your Name"           # if not set globally
git config user.email "you@example.com"

git add .gitignore LICENSE
git commit -m "Initial commit: gitignore and license"

git remote add origin git@github.com:<username>/devops-fundamentals.git
git push -u origin main
```

## Everyday portfolio-repo git commands

```bash
git status                       # what's staged/unstaged/untracked
git add scripts/day15-tool/       # stage one script folder at a time
git commit -m "Add disk usage report script"
git log --oneline --graph        # inspect commit history readability
git diff --cached                # review exactly what a commit will contain, before committing
git rm --cached path/to/file     # unstage/untrack a file (keeps it locally) — e.g. an accidentally-added .env
git ls-files | grep -i secret     # sanity-check nothing sensitive is tracked
```

## .gitignore template

```gitignore
# Secrets and local config
.env
.env.*
*.pem
*.key
credentials.json
kubeconfig*
.aws/
.terraform/
*.tfvars

# Language/runtime cruft
__pycache__/
*.pyc
venv/
.venv/
node_modules/

# Build output
site/

# OS/editor noise
.DS_Store
.vscode/
```

## Top-level README skeleton

```markdown
# devops-fundamentals

![CI](https://github.com/<user>/devops-fundamentals/actions/workflows/lint.yml/badge.svg)
![License](https://img.shields.io/github/license/<user>/devops-fundamentals)

One-line description of what this repo is.

## What this demonstrates
- Skill/tool -> link to relevant scripts/ subfolder

## Table of contents
- [Getting started](#getting-started)
- [Scripts](#scripts)
- [Docs site](#docs-site)

## Getting started
\`\`\`bash
git clone https://github.com/<user>/devops-fundamentals.git
\`\`\`

## Docs site
https://<user>.github.io/devops-fundamentals/
```

## Per-script README skeleton

```markdown
# <script-name>

**What it does:** one or two sentences.

**Why it exists:** the real problem/day this came from.

## Usage
\`\`\`bash
python3 script.py --flag value
\`\`\`

## Prerequisites
- Python 3.x, `pip install <deps>`
- AWS credentials configured via a named profile (never hardcoded)
```

## MkDocs commands

```bash
pip install mkdocs mkdocs-material   # install
mkdocs new .                          # scaffold mkdocs.yml + docs/index.md
mkdocs serve                          # local preview, http://127.0.0.1:8000, live reload
mkdocs build                          # render static site to ./site/
mkdocs gh-deploy --force              # build + push to gh-pages branch (GitHub Pages)
```

## Minimal mkdocs.yml

```yaml
site_name: DevOps Fundamentals
site_url: https://<username>.github.io/devops-fundamentals/
repo_url: https://github.com/<username>/devops-fundamentals
theme:
  name: material
nav:
  - Home: index.md
  - Linux: linux/day01-filesystem.md
  - AWS: aws/day16-ec2-tagging.md
markdown_extensions:
  - toc:
      permalink: true
  - codehilite
```

## GitHub Pages basics

```
Settings → Pages → Build and deployment → Source: Deploy from a branch
Branch: gh-pages  /  (root)
```

- `mkdocs gh-deploy` creates/updates the `gh-pages` branch automatically — you only set the Pages source once.
- Site URL pattern: `https://<username>.github.io/<repo-name>/`.
- Re-run `mkdocs gh-deploy` after any docs change; it's safe to run repeatedly (force-pushes `gh-pages` only, never touches `main`).

## Secret-scanning quick checks

```bash
grep -RniE "aws_secret|AKIA[0-9A-Z]{16}|BEGIN (RSA|OPENSSH) PRIVATE KEY" .
pip install detect-secrets && detect-secrets scan
# or: gitleaks detect --source .
```
