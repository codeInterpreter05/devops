# Day 21 — Portfolio & Documentation: Building Your Portfolio Repo

**Phase:** 0 – Foundation | **Week:** W3 | **Domain:** Review | **Flag:** 📌

## Brief

A resume line that says "learned Linux, AWS, Docker, Kubernetes, Terraform, CI/CD" is a claim. A public GitHub repo with working scripts, clear READMEs, and a commit history spanning weeks is *evidence*. Hiring managers and technical interviewers routinely open a candidate's GitHub profile before or during a screen — it's one of the few free, low-effort signals they have that isn't self-reported. A well-built portfolio repo answers three questions before anyone asks them out loud: can this person actually write and run scripts, do they document their work well enough for someone else (or future-them) to use it, and do they follow basic engineering hygiene (git discipline, no leaked secrets, a sane repo layout)?

This is not a new deep-technical day — it's the day you package everything from Days 1–20 into something a stranger can look at in five minutes and come away thinking "this person does real work." Today's second file covers the writing/branding half of that same goal.

This day is split into two focused files:

1. **This file** — repo structure, top-level README craft, git hygiene, and MkDocs for a browsable docs site.
2. **[02-README-Technical-Writing-And-Personal-Branding.md](02-README-Technical-Writing-And-Personal-Branding.md)** — writing a technical blog post and updating LinkedIn to match the work you've actually done.

## Repo layout conventions

A portfolio repo needs a layout that's obvious to a stranger within seconds of landing on it. Two common patterns, both defensible:

**Mirror-the-roadmap layout** — one folder per day/topic, useful if the repo *is* your visible learning log:

```
devops-fundamentals/
├── README.md
├── LICENSE
├── .gitignore
├── mkdocs.yml
├── docs/
│   ├── index.md
│   └── ...              # mirrors scripts/ folders, one page per topic
└── scripts/
    ├── day15-disk-usage-report/
    │   ├── disk_usage_report.py
    │   └── README.md
    ├── day16-ec2-tagging/
    │   ├── tag_untagged_ec2.py
    │   └── README.md
    ├── day17-k8s-manifest-generator/
    │   ├── render_manifests.py
    │   ├── templates/
    │   └── README.md
    ├── day19-strace-perf-fix/
    │   ├── slow_script_before.sh
    │   ├── fixed_script_after.sh
    │   └── README.md
    └── day20-troubleshooting-notes/
        └── README.md
```

**Topic-grouped layout** — folders named by function (`aws/`, `kubernetes/`, `linux/`, `ci-cd/`) instead of day number, better once the repo outgrows "learning log" and needs to read as a general-purpose toolbox. Either is fine; what's *not* fine is a flat pile of unrelated scripts with no grouping and no per-script README, because a reviewer then has to open every file to figure out what anything does.

Whichever you choose, keep it consistent and put a one-paragraph `README.md` inside every script's subfolder: what it does, why it exists, how to run it, and any prerequisites. A script with no usage instructions signals "I wrote this once and never touched it again" — the opposite of what you want a portfolio to say.

## Writing an effective top-level README

The top-level `README.md` is the single most-read file in the repo — most visitors never click past it. What actually gets read:

- **A one-line description** immediately under the title — what this repo *is*, in plain language, not "a collection of scripts."
- **Badges** — cheap to add, signal that the repo is maintained and (if CI is wired up) actually passes checks.
- **A "What this demonstrates" section** — explicitly map the repo's contents to skills, because recruiters skim, they don't infer.
- **A table of contents** for anything with more than ~5 sections, so a reviewer can jump straight to what they care about.
- **Screenshots or sample output** wherever a script produces something visual (a terminal capture of a report, a rendered Kubernetes manifest, a Grafana panel) — this is often the only part of a README a busy reviewer actually looks at closely.

Concrete skeleton:

```markdown
# DevOps Fundamentals

![CI](https://github.com/<username>/devops-fundamentals/actions/workflows/lint.yml/badge.svg)
![License](https://img.shields.io/github/license/<username>/devops-fundamentals)
![Last commit](https://img.shields.io/github/last-commit/<username>/devops-fundamentals)

A collection of working scripts, configs, and notes built while completing a
100-day DevOps roadmap — Linux internals, AWS automation, Kubernetes,
Terraform, CI/CD, and observability. Every script here runs; every README
explains why it exists and how to use it.

## What this demonstrates

- **Linux & scripting** — `scripts/day15-disk-usage-report/`, a Python CLI
  that reports top disk consumers and exits non-zero above a threshold
  (cron/alerting-friendly).
- **AWS automation** — `scripts/day16-ec2-tagging/`, a boto3 script that
  finds and tags untagged EC2 instances against a naming policy.
- **Templating / Kubernetes** — `scripts/day17-k8s-manifest-generator/`,
  Jinja2-rendered manifests from a single values file.
- **Debugging & performance** — `scripts/day19-strace-perf-fix/`, a
  strace-profiled fix for a script that was silently blocking on I/O.

## Table of contents

- [Repo layout](#repo-layout)
- [Getting started](#getting-started)
- [Scripts](#scripts)
- [Docs site](#docs-site)

## Getting started

\`\`\`bash
git clone https://github.com/<username>/devops-fundamentals.git
cd devops-fundamentals/scripts/day15-disk-usage-report
python3 disk_usage_report.py --threshold 80
\`\`\`

## Docs site

Full write-ups for every topic are published at
https://<username>.github.io/devops-fundamentals/ (built with MkDocs).
```

Keep the top-level README focused on orientation and links — detailed explanations belong in each script's own README or in the MkDocs site, not crammed into the front page.

## Git hygiene for a portfolio repo

A messy commit history (`fix`, `fix2`, `asdf`, `final final v3`) undercuts the exact impression you're trying to create. Aim for commits that describe *why*, not just *what*:

```bash
git init
git add README.md .gitignore LICENSE
git commit -m "Initial commit: repo skeleton and README"

git add scripts/day15-disk-usage-report/
git commit -m "Add disk usage report script with threshold alerting"

git branch -M main
git remote add origin git@github.com:<username>/devops-fundamentals.git
git push -u origin main
```

Squash noisy work-in-progress commits before pushing to `main` if your local history is messy (`git rebase -i`), but never rewrite history that's already been pushed and might be shared — treat `main` as append-only once public.

**Secrets are the single biggest risk in a public repo.** Day 16 covers IAM users, access keys, and least-privilege policy design — the same discipline applies here in reverse: never let an access key, `.pem` file, `.env`, or kubeconfig with embedded credentials end up in a commit. A minimal `.gitignore`:

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

# OS/editor noise
.DS_Store
.vscode/
```

Add a pre-commit scan (`pip install detect-secrets` or `gitleaks`) so a credential never reaches a commit in the first place — catching it after `git push` is too late, because the moment something is pushed to a public repo you must treat it as compromised: rotate/revoke the credential immediately (see Day 16), don't just delete the file in a follow-up commit (it's still in history and in anyone's already-cloned copy).

## Using MkDocs to turn Markdown into a browsable site

A folder of `.md` files on GitHub is readable, but a proper docs *site* — searchable, navigable, with a real table of contents — reads as more polished and is a good excuse to demonstrate static-site tooling. [MkDocs](https://www.mkdocs.org/) turns a `docs/` folder of Markdown into a static site with almost no config.

```bash
pip install mkdocs mkdocs-material
mkdocs new .              # scaffolds mkdocs.yml + docs/index.md
```

Minimal `mkdocs.yml`:

```yaml
site_name: DevOps Fundamentals
site_url: https://<username>.github.io/devops-fundamentals/
repo_url: https://github.com/<username>/devops-fundamentals
theme:
  name: material
  palette:
    scheme: slate
nav:
  - Home: index.md
  - Linux:
      - Filesystem hierarchy: linux/day01-filesystem.md
      - Subprocess & shell execution: linux/day15-subprocess.md
  - AWS:
      - EC2 tagging automation: aws/day16-ec2-tagging.md
  - Kubernetes:
      - Manifest templating: k8s/day17-manifests.md
markdown_extensions:
  - toc:
      permalink: true
  - codehilite
```

```bash
mkdocs serve              # http://127.0.0.1:8000, live-reloads on save
mkdocs build              # renders static site to ./site/
mkdocs gh-deploy --force  # builds + pushes the site to the gh-pages branch
```

`mkdocs gh-deploy` handles the whole GitHub Pages publish step for you — it builds the site and force-pushes it to a `gh-pages` branch. After the first run, enable Pages in the repo's Settings → Pages, set the source branch to `gh-pages`, and the site is live at `https://<username>.github.io/<repo>/` within a minute or two. Re-run `mkdocs gh-deploy` any time the docs change; it's safe to run repeatedly.

## Points to Remember

- Every script subfolder needs its own README (what it does, how to run it, prerequisites) — the top-level README is for orientation and linking, not exhaustive detail.
- Badges, a "what this demonstrates" section, and a table of contents are what actually get read on a repo's front page; put your best material there, not buried three folders deep.
- Treat any pushed secret as compromised the instant it's pushed — rotate/revoke (Day 16 IAM material), don't just delete the file in a later commit.
- `.gitignore` should be written *before* your first commit, not reactively after you notice a `.env` file got staged.
- `mkdocs serve` for local iteration, `mkdocs gh-deploy` to publish — no separate CI/CD pipeline required to get a working docs site live.

## Common Mistakes

- Committing AWS credentials, `.pem` keys, kubeconfigs, or `.env` files to a public repo — the most common and most damaging portfolio mistake; assume anything ever pushed publicly is compromised, even after deletion.
- A repo with no top-level README, or one that just says "my devops scripts" with no structure, no links, and no explanation of what any individual script does.
- Scripts with no comments and no usage instructions — a reviewer has to read the entire source just to learn what arguments it takes.
- One giant "final commit" with everything added at once — an empty or fabricated-looking commit history is almost as bad as a messy one; it suggests the repo was assembled for show rather than built incrementally.
- Forgetting to actually enable GitHub Pages after running `mkdocs gh-deploy` (the `gh-pages` branch exists, but Pages is still pointed at "None" in repo settings) — the site silently 404s.
