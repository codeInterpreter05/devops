# Day 8 — Resources: Git Deep Dive I

## Primary (assigned)

- **Pro Git Book, Ch. 1–3** (free, git-scm.com/book) — the assigned starting point for this day. Chapter 1 covers what version control and Git actually are; Chapter 2 is hands-on basics (init, add, commit, log, remotes); Chapter 3 is branching and merging in depth, including the branch-as-a-pointer mental model this whole day builds on.

## Deepen your understanding

- **Pro Git Book, Ch. 10 — Git Internals** (git-scm.com/book/en/v2/Git-Internals-Plumbing-and-Porcelain) — the direct continuation of today's internals topic, straight from the source: plumbing vs. porcelain, the object model, refs, and packfiles, in more depth than any single day's notes can cover.
- **"Git from the Bottom Up" by John Wiegley** (jwiegley.github.io/git-from-the-bottom-up) — a free, short, dense walkthrough that builds Git's object model up from first principles rather than describing it top-down. The best free complement to today's internals file if the "blobs/trees/commits" explanation didn't fully click yet.
- **Atlassian Git Tutorial — Merging vs. Rebasing** (atlassian.com/git/tutorials/merging-vs-rebasing) — a clear, diagram-heavy treatment of exactly today's interview question, including the "golden rule of rebasing" framed for team workflows rather than solo use.
- **Julia Evans — "Oh Shit, Git!?!" (ohshitgit.com)** — a blunt, practical reference for exactly the disaster-recovery scenarios in today's lab (bad merge, lost commits, wrong branch) — good to bookmark for real incidents, not just study.

## Reference

- **git-scm.com/docs** — the official command reference. `git-rebase(1)`, `git-reflog(1)`, and `git-cherry-pick(1)` are the three worth reading end-to-end after today's labs; each documents edge cases (like `--onto`, reflog expiry config, and `-x`/`-n`) beyond what a single day's notes can hold.
- **`git help <command>`** — every reference doc above is also installed locally; no internet needed once Git itself is installed.

## Practice

- **Learn Git Branching** (learngitbranching.js.org) — an interactive, visual sandbox purpose-built for practicing exactly today's topics: rebase, cherry-pick, and reading the commit graph, with instant visual feedback per command. The single best hands-on complement to today's LAB.md.
- **Git Graph — VS Code extension** (`mhutchie.vscode-git-graph`, marketplace ID `mhutchie.vscode-git-graph`) — the tool named for today: renders the commit DAG interactively inside the editor, which makes `git log --graph --oneline` output much easier to explore visually once a history gets busy (multiple branches, several rebases).
- **Oh My Git!** (ohmygit.org) — a free, open-source game that teaches Git's internals and commands (including rebase and reflog-style recovery) through puzzles that visualize the commit graph in real time as you act on it.
