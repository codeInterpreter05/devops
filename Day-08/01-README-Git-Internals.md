# Day 8 — Git Deep Dive I: Git Internals

**Phase:** 0 – Foundation | **Week:** W2 | **Domain:** Git | **Flag:** ⚡ Interview-critical

## Brief

Most people learn Git as a sequence of memorized incantations: `add`, `commit`, `push`, panic, Stack Overflow. That works until something goes sideways — a botched rebase, a "lost" commit after a hard reset, a merge conflict that makes no sense — and then you need the mental model underneath, not another memorized command. Git is not a magic version-tracking black box; it's a surprisingly simple **content-addressable key-value store** with a thin layer of porcelain on top. Once you see that, `git log --graph`, rebase, cherry-pick, and reflog recovery (the rest of today) stop being separate tricks and become obvious consequences of one idea.

This day is split into three focused files:

1. **This file** — what Git actually is, the object model (blobs/trees/commits/refs), the `.git` directory, and plumbing vs. porcelain.
2. **[02-README-Rebase-And-Cherry-Pick.md](02-README-Rebase-And-Cherry-Pick.md)** — interactive rebase mechanics, rebase vs. merge, and cherry-pick.
3. **[03-README-Reflog-And-Recovery.md](03-README-Reflog-And-Recovery.md)** — the reflog, recovering from disasters, and true data loss.

## Git is a content-addressable object store

Strip away branches, commits, and the working directory, and Git is just a directory (`.git/objects`) full of files, each named after the **SHA-1 hash of its own content**. Store the same content twice — even in two totally unrelated files, commits, or projects — and Git stores it exactly once, because identical content hashes to the same name. This is why Git is fast at detecting "this blob already exists" and why cloning a repo doesn't balloon in size every time a file is renamed but unchanged (the blob content is identical, only the tree entry pointing to it changes).

There are four object types, and every one of them is content-addressed the same way:

| Object | What it stores | Analogy |
|---|---|---|
| **blob** | Raw file *contents* — no filename, no permissions, just bytes | A file's data, anonymized |
| **tree** | A directory listing: entries of `(mode, type, sha, name)` pointing to blobs or other trees | A directory snapshot |
| **commit** | A pointer to one tree, pointer(s) to parent commit(s), author, committer, timestamp, message | A snapshot + metadata + history link |
| **tag (annotated)** | A pointer to another object (usually a commit) plus a message, tagger, signature | A labeled bookmark with metadata |

Critically: **a commit does not store a diff.** It stores a pointer to a complete tree (a full snapshot of the entire project at that point). Git computes diffs on the fly by comparing two trees when you ask for one (`git diff`, `git log -p`). This is why operations like `git checkout <commit>` are just "point HEAD at this tree and check it out" rather than "replay N patches" — and why rebase (replaying commits one at a time, covered in file 2) is comparatively expensive: it has to reconstruct a diff for each commit and reapply it.

## Walking the object graph by hand

```bash
git init demo && cd demo
echo "hello" > file.txt
git add file.txt
git commit -m "first commit"
```

Now inspect what actually happened:

```bash
git cat-file -t HEAD          # -> commit
git cat-file -p HEAD          # -> tree <sha>, author, committer, message
git cat-file -t <tree-sha>    # -> tree
git cat-file -p <tree-sha>    # -> 100644 blob <sha>  file.txt
git cat-file -t <blob-sha>    # -> blob
git cat-file -p <blob-sha>    # -> hello
```

`git cat-file -p` (pretty-print) decodes any object by its hash regardless of type — this is the single most useful command for building intuition about internals. Chase the chain manually once (commit → tree → blob) and the "blobs/trees/commits" diagram in every Git tutorial stops being abstract.

You can also create objects directly, bypassing the working directory entirely:

```bash
echo "hello" | git hash-object --stdin       # prints the sha it WOULD have, doesn't store
echo "hello" | git hash-object -w --stdin    # -w = actually write it into .git/objects
git ls-tree HEAD                              # list a tree's entries
git ls-tree -r HEAD                           # recursive — show blobs in subdirectories too
```

`git hash-object` proves the point: the hash is a pure function of content plus a type header (`"blob 6\0hello\n"` in this case, roughly), not of when, where, or why you created it. Two developers on opposite ends of the earth who happen to write byte-identical file contents produce the identical blob SHA.

## Refs: names pointing at commits

A **ref** is nothing more than a file containing a 40-character SHA-1 (or a pointer to another ref). Branches, tags, and `HEAD` are all refs:

```bash
cat .git/HEAD                    # ref: refs/heads/main   (a SYMBOLIC ref, most of the time)
cat .git/refs/heads/main          # <sha-of-latest-commit-on-main>
git rev-parse HEAD                # resolves HEAD -> refs/heads/main -> the actual sha
git rev-parse main                # same sha, via the branch name directly
```

- `HEAD` is normally a **symbolic ref** — a pointer to a pointer (`ref: refs/heads/main`). This is what makes committing "just work": commit, and Git updates whatever branch `HEAD` points to.
- In **detached HEAD** state, `HEAD` points directly at a commit SHA instead of at a branch. Commits made here aren't on any branch — they're one `git checkout` away from becoming unreachable (relevant in file 3).
- Creating a branch (`git branch foo`) is *just writing a new 41-byte file* (`refs/heads/foo`) containing the current commit's SHA. This is why branching in Git is instant and cheap regardless of repo size — it's not copying anything.
- Modern repos also keep a `packed-refs` file (refs get "packed" for efficiency instead of one loose file each) and use a reflog per ref (covered in depth in file 3).

## Tour of `.git`

```
.git/
├── HEAD            # symbolic ref: which branch you're on
├── config           # repo-local config (remotes, user overrides, etc.)
├── index            # the staging area — a binary file listing what's staged for next commit
├── objects/          # every blob, tree, commit, tag — the actual content-addressable store
│   ├── ab/          # objects are stored as objects/<first 2 hex chars>/<remaining 38 chars>
│   └── pack/         # compacted "pack files" — many loose objects merged + delta-compressed
├── refs/
│   ├── heads/        # local branches
│   └── tags/         # tags
└── logs/             # the reflog — HEAD's and each branch's history of ref movements (file 3)
```

The **index** (staging area) deserves a callout: `git add` doesn't touch a commit or a branch — it writes a blob object and updates the index to reference it. `git commit` then builds a tree from whatever the index currently says, and wraps it in a commit object. This is the entire reason a "staged" state exists as distinct from "working directory" and "last commit": the index is Git's scratch space for *composing the next tree*.

## `git log --graph --oneline`: the commit DAG made visible

Because every commit stores its parent's SHA(s), the full commit history is a **directed acyclic graph (DAG)** — commits point backward at their parents, merge commits have two (or more) parents, and there's no notion of a single "next" commit (that's what branches/refs are for: bookmarks into the DAG).

```bash
git log --graph --oneline --decorate --all
```

- `--graph` draws the ASCII graph structure (`|`, `*`, `\`, `/`) showing where branches diverged and merged.
- `--oneline` collapses each commit to `<short-sha> <subject>` instead of the full 6-line format.
- `--decorate` labels commits with the branch/tag names (refs) currently pointing at them.
- `--all` shows every ref, not just the current branch's ancestry — essential for seeing a feature branch that hasn't been merged yet.

Reading this output *is* reading the object graph: each `*` is a commit object, each line connecting two `*`s is a parent pointer, and a commit with two parent lines converging into it is a merge commit. This single command is how you sanity-check "did my rebase actually do what I think," "is this branch actually merged," or "why does this history look like spaghetti" — all questions you'll ask constantly once you start doing the rebase/cherry-pick work in file 2.

A commit with two parents is unambiguous proof of a merge; a linear chain with no forks is proof that history was rebased or fast-forwarded rather than merged. Being able to read that at a glance from `--graph` output, cold, is a strong interview signal.

The **VS Code Git Graph extension** (`mhutchie.vscode-git-graph`) renders this same DAG interactively — clickable commits, drag-to-compare, visual branch/merge lines — and is worth installing if you're spending real time untangling history rather than just reading it linearly in a terminal.

## Plumbing vs. porcelain

Git's own documentation draws this distinction explicitly:

- **Porcelain** — the user-facing commands you use daily: `git commit`, `git branch`, `git merge`, `git rebase`, `git log`, `git status`. Designed for humans, with friendly output and safety rails.
- **Plumbing** — the low-level commands porcelain is built from: `git cat-file`, `git hash-object`, `git write-tree`, `git commit-tree`, `git ls-tree`, `git rev-parse`, `git update-ref`, `git rev-list`. Designed for scripts and for building other tools on top of Git — stable, scriptable output, no interactive prompts.

Every porcelain command is, underneath, a sequence of plumbing calls. `git commit` roughly does: take the index, `write-tree` it into a tree object, `commit-tree` that tree with the current `HEAD` as parent to produce a commit object, then `update-ref refs/heads/<branch>` (or wherever `HEAD` points) to the new commit SHA. Knowing this is what makes advanced recovery (file 3) and advanced history surgery (file 2) tractable instead of terrifying — when porcelain commands don't do quite what you need, plumbing always can, because everything porcelain does is just plumbing with guardrails.

## Points to Remember

- Git stores four object types — blob, tree, commit, (annotated) tag — each named by the SHA of its own content. Identical content is always stored once, regardless of filename or location.
- A commit points to a tree (a full snapshot) plus parent commit(s), not a diff. Diffs are computed on demand by comparing trees.
- Refs (branches, tags, `HEAD`) are just files containing a SHA (or, for `HEAD`, usually a pointer to another ref). Creating a branch is nearly free — it's writing one small file, not copying the repo.
- The index (`.git/index`) is the staging area: `git add` writes a blob and updates the index; `git commit` turns the index into a tree, then wraps it in a commit.
- `git log --graph --oneline --decorate --all` visualizes the commit DAG directly — a merge commit has two parent lines converging; a straight line means fast-forward or rebase, not merge.
- Plumbing commands (`cat-file`, `hash-object`, `rev-parse`, etc.) are the primitives porcelain commands (`commit`, `merge`, `rebase`) are built from — reach for them when you need to inspect or repair a repo at a level porcelain doesn't expose.

## Common Mistakes

- Thinking a commit stores "the changes" (a diff) rather than a full tree snapshot — this misunderstanding is what makes rebase/cherry-pick conflict resolution feel mysterious instead of mechanical (see file 2).
- Assuming branches are expensive or "heavy" (a habit carried over from other VCS tools) and avoiding branching as a result — in Git a branch is a 41-byte file, not a repo copy.
- Confusing the working directory, the index (staging area), and `HEAD`/the last commit as one blurry "current state" — they're three genuinely distinct snapshots, which is exactly what `git status` and `git diff` (no args) vs. `git diff --staged` are reporting on.
- Never having run `git cat-file -p` on anything — it's the fastest way to stop treating Git objects as opaque and start reading them directly when something looks wrong.
- Reading `git log --graph` output without `--all` and concluding a branch "doesn't exist" or "wasn't committed" just because it's not visible in the current branch's ancestry.
