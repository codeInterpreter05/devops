# Day 21 — Lab: Portfolio & Documentation

**Goal:** Publish a real, public `devops-fundamentals` GitHub repository with at least 5 working scripts from this phase, each documented, plus a browsable MkDocs site, a technical blog post draft, and an updated LinkedIn profile — the tangible artifact that anchors the rest of your job search.

**Prerequisites:** A GitHub account, `git` installed and configured (`git config --global user.name "Your Name"` / `user.email`), Python 3 + `pip` for MkDocs, and the scripts/notes you've produced across Days 1–20 of this phase (substitute your own equivalents if your exact scripts differ from the examples below).

---

### Lab 1 — Initialize the devops-fundamentals repo

1. Create the repo on GitHub (empty, no README/license yet, so you control the first commit) named `devops-fundamentals`.
2. Locally:
   ```bash
   mkdir devops-fundamentals && cd devops-fundamentals
   git init
   git branch -M main
   ```
3. Add a `.gitignore` before anything else goes in (see CHEATSHEET.md for the template) and a `LICENSE` (MIT is a reasonable default for a learning portfolio — pick one at [choosealicense.com](https://choosealicense.com)).
4. Commit and push:
   ```bash
   git add .gitignore LICENSE
   git commit -m "Initial commit: gitignore and license"
   git remote add origin git@github.com:<username>/devops-fundamentals.git
   git push -u origin main
   ```

**Success criteria:** The repo exists on GitHub, is public, has a `.gitignore` and `LICENSE` in its first commit(s), and you can `git clone` it fresh into a different directory without errors.

---

### Lab 2 — Select and organize at least 5 scripts

This is the assigned hands-on activity for today — do it for real, not just read about it.

1. Pick at least 5 scripts/artifacts you produced in this phase. If you don't have 5 exact matches, substitute your own — the point is 5 *real, runnable* pieces of work, not placeholders. Reasonable candidates:
   - A disk-usage reporting/CLI tool (Day 15).
   - An AWS EC2 tagging or inventory automation script (Day 16).
   - A Jinja2-templated Kubernetes manifest generator (Day 17).
   - A performance fix you diagnosed with `strace` (Day 19) — include the before/after scripts.
   - Troubleshooting notes or a runbook from a debugging exercise (Day 20).
2. For each, create a subfolder under `scripts/` (e.g. `scripts/day15-disk-usage-report/`) and copy the script(s) in.
3. Test each script actually runs from a clean checkout before committing it — a broken script in a portfolio repo is worse than no script.
4. Write a per-script `README.md` in each subfolder covering: what it does, why it exists, how to run it (exact command), and any prerequisites (e.g. `pip install boto3`, AWS credentials configured via a named profile — never hardcoded).
5. Commit each script with its own descriptive commit, not one giant "add all scripts" commit:
   ```bash
   git add scripts/day15-disk-usage-report/
   git commit -m "Add disk usage report script with threshold alerting"
   ```

**Success criteria:** At least 5 script subfolders exist under `scripts/`, each with a working script and its own README, each added in a separate, clearly-described commit, and `git log --oneline` tells a coherent story of the repo being built incrementally.

---

### Lab 3 — Write the top-level README

1. Using the template in `01-README-Building-Your-Portfolio-Repo.md`, write `README.md` at the repo root: one-line description, badges, a "What this demonstrates" section mapping folders to skills, a table of contents, and a "Getting started" snippet that actually works if someone clones and runs it.
2. Add at least one screenshot or captured terminal output (a `.png` under an `assets/` or `docs/img/` folder, or a fenced code block showing real output) for your most visually interesting script.
3. Push it and view the rendered README on GitHub's repo page — check that code blocks, tables, and links render correctly (GitHub's Markdown renderer is close to but not identical to some editors' preview).

**Success criteria:** Someone who has never seen the repo before can read the top-level README in under two minutes and correctly state what the repo contains and what skills it's meant to demonstrate.

---

### Lab 4 — Set up MkDocs and deploy to GitHub Pages

1. Install and scaffold:
   ```bash
   pip install mkdocs mkdocs-material
   mkdocs new .
   ```
2. Move or write one Markdown page per topic under `docs/` (can start by copying/adapting content from your Day 1–20 README files), and wire up `nav:` in `mkdocs.yml` (see CHEATSHEET.md for a minimal example).
3. Preview locally:
   ```bash
   mkdocs serve
   ```
   Confirm it's reachable at `http://127.0.0.1:8000` and navigation works.
4. Deploy:
   ```bash
   mkdocs gh-deploy --force
   ```
5. In the GitHub repo's Settings → Pages, confirm the source is set to the `gh-pages` branch, then visit `https://<username>.github.io/devops-fundamentals/` and confirm the live site matches your local preview.

**Success criteria:** The MkDocs site is live at a public GitHub Pages URL, navigation matches `mkdocs.yml`, and the URL is linked from your top-level README.

---

### Lab 5 — Write one technical blog post draft

1. Pick one concept from this phase you can explain confidently without notes — the narrower and more specific, the better (a single bug you fixed beats a broad topic survey).
2. Draft it using the four-part structure from `02-README-Technical-Writing-And-Personal-Branding.md`: problem/context, explanation, concrete example (real commands/output), takeaway.
3. Write it as a Markdown file first (in `docs/blog/` or a scratch file) so you can reuse it on your MkDocs site regardless of where else you publish it.
4. Draft the outline/skeleton at minimum — a full polished post is ideal, but a complete outline with real code blocks filled in for each section satisfies today's activity.

**Success criteria:** You have a Markdown draft with all four sections filled in (not just headers), including at least one real, tested command/output block — not hypothetical pseudocode.

---

### Lab 6 — Update LinkedIn

Work through this checklist on your actual LinkedIn profile:

- [ ] **Skills section** — add/reorder skills to match tools you've actually used hands-on this phase (e.g. `Linux`, `Bash`, `AWS`, `Docker`, `Kubernetes`, `Terraform`, `Git`, `CI/CD`) — remove or deprioritize vague soft-skill tags if your list is cluttered.
- [ ] **Featured section** — add a link to your `devops-fundamentals` GitHub repo with a descriptive custom title.
- [ ] **Featured section** — add a link to your blog post (once published) or your MkDocs site.
- [ ] **About summary** — rewrite it to name specific tools and link to your portfolio repo, following the concrete-vs-generic contrast in file 2. Read it out loud once — if it sounds like it could describe anyone, rewrite it.
- [ ] **Headline** — confirm it reflects your target role/stack rather than a default job title, if applicable.

**Success criteria:** A stranger viewing your LinkedIn profile can click through to your GitHub repo and blog post within one click from the profile itself, and your About section names at least 3 concrete tools/projects rather than only adjectives.

---

### Cleanup

Before considering the repo "published," treat it as if it's already being reviewed by a hiring manager:

```bash
# Re-check the full diff of everything staged before any push
git diff --cached

# Search tracked files for likely secret patterns
grep -RniE "aws_secret|AKIA[0-9A-Z]{16}|BEGIN (RSA|OPENSSH) PRIVATE KEY|password\s*=" .

# Confirm nothing sensitive is tracked that should be ignored
git ls-files | grep -iE "\.env|\.pem|credentials|kubeconfig"
```

If anything sensitive was ever committed and pushed — even briefly — rotate or revoke that credential immediately (IAM keys via the AWS console/CLI, per Day 16) rather than relying on deleting the file in a later commit; a public repo's history (and any fork/clone made before your fix) must be assumed compromised the moment it's pushed.

### Stretch challenge

Add a GitHub Actions workflow (`.github/workflows/lint.yml`) that runs on every push and pull request, installs `ruff` and `mypy` (from Day 15), and lints/type-checks every Python script under `scripts/`. Wire the badge from this workflow into your top-level README so it shows a live pass/fail status — this alone demonstrates CI/CD literacy on top of everything else in the repo.
