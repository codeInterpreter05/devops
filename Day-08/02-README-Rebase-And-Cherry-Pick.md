# Day 8 — Git Deep Dive I: Rebase & Cherry-Pick

**Phase:** 0 – Foundation | **Week:** W2 | **Domain:** Git | **Flag:** ⚡ Interview-critical

## Brief

"Merge or rebase?" is one of the most reliably asked Git interview questions, precisely because a shallow answer ("rebase makes history cleaner") reveals someone who's memorized a rule of thumb, not someone who's actually hit a rebase conflict on a shared branch and had to explain it to a teammate. Both commands solve the same underlying problem — integrating divergent history — but they do it via genuinely different mechanics, with different consequences for anyone else who has a copy of that history. This file covers those mechanics in enough depth to answer that question with real conviction, plus cherry-pick, which is the same replay machinery applied to a single commit.

## What merge actually does

`git merge feature` (while on `main`) finds the common ancestor of `main` and `feature`, then creates a **new commit with two parents**: the current tip of `main` and the tip of `feature`. Nothing about the existing commits on either branch changes — their SHAs, parents, and content are untouched. The merge commit is purely additive: it's a new node in the DAG that has two incoming parent edges (see file 1's `--graph` discussion).

```bash
git checkout main
git merge feature
# fast-forward if main hasn't moved since feature branched (no merge commit at all)
# otherwise: a real merge commit with two parents, e.g. "Merge branch 'feature'"
```

If `main` hasn't advanced since `feature` branched off it, Git defaults to a **fast-forward**: it just moves the `main` ref forward to `feature`'s tip, since a merge commit wouldn't add any information. Force a real merge commit even when a fast-forward is possible with `git merge --no-ff` — many teams do this on purpose so "this was a feature branch" stays visible in history.

**Merge is non-destructive.** Existing commits are never rewritten, so it's always safe on shared/pushed branches — nobody else's history becomes invalid.

## What rebase actually does

`git rebase main` (while on `feature`) does something structurally different: it finds the same common ancestor, then **replays each of `feature`'s commits, one at a time, as brand-new commits on top of `main`'s current tip**. "Replay" here means: take the diff that commit introduced, apply it to the new parent, and create a new commit object with that resulting tree and a new parent pointer — the message and author are preserved, but **the resulting commit has a different SHA**, because a commit's hash includes its parent pointer, and the parent has changed.

```bash
git checkout feature
git rebase main
# feature's commits are gone (as objects with their old SHAs) and replaced by
# new commits with the same diffs/messages but new SHAs, sitting on top of main's tip
```

The old commits aren't immediately destroyed — they become **unreferenced** (no branch or tag points at them anymore), which is exactly what the reflog exists to help you recover from if the rebase goes wrong (file 3). But as far as any *branch* is concerned, that history has been replaced, not preserved.

Because rebase replays commits **one at a time**, conflicts are resolved per-commit rather than in one big merge conflict: if commit 3 of 5 conflicts, you resolve just that one, `git add` the result, then `git rebase --continue`, and Git moves on to replaying commit 4 against the now-resolved state. This is genuinely more work than a merge conflict when many of your commits touch the same lines, but it also means each conflict is scoped to a single, small, understandable change — often easier to reason about than one giant merge diff.

```bash
git rebase --continue   # after resolving the current commit's conflict and `git add`-ing it
git rebase --skip       # drop this commit entirely, move to the next
git rebase --abort      # bail out completely, restore feature to exactly where it was before rebase started
```

## The golden rule of rebasing

**Never rebase commits that other people have already pulled/based work on.** Because rebase produces new SHAs for every replayed commit, anyone who already has the old commits now has history that has diverged from yours in a way Git can't reconcile automatically — their next `git pull` either creates a confusing merge of "old" and "new" versions of the same logical commits, or requires them to know to `git pull --rebase` /reset onto your rewritten history themselves. Rewriting history you've already shared doesn't just create noise, it can silently duplicate every commit in the branch's history for anyone who merges instead of resetting.

The practical rule: rebase freely on **local, unpushed, solely-yours** branches to clean up work-in-progress before it becomes visible to anyone else. Once you've pushed and especially once someone else might have pulled or branched from it, prefer merge (or, if a team convention truly requires a linear history on a shared branch, coordinate an explicit force-push and make sure everyone knows to reset, not merge).

## Interactive rebase: `git rebase -i <base>`

Interactive rebase is the same replay mechanism, but it opens an editor listing every commit between `<base>` and your current `HEAD` (oldest first) and lets you choose what happens to each one before it's replayed:

```bash
git rebase -i HEAD~5      # edit the last 5 commits on the current branch
git rebase -i main         # edit everything since this branch diverged from main
```

```
pick   a1b2c3d Add login form
squash e4f5g6h Fix typo in login form
fixup  h7i8j9k Remove console.log
reword k0l1m2n Add password validation
edit   n3o4p5q Add remember-me checkbox
drop   q6r7s8t Revert accidental debug commit
```

| Verb | Effect |
|---|---|
| `pick` | Keep the commit as-is |
| `reword` | Keep the commit's changes, but stop to let you edit the commit message |
| `edit` | Stop right after applying this commit, dropping you into a shell to amend it (add files, split it, etc.) before continuing |
| `squash` | Combine this commit's changes into the **previous** one, and prompt to merge both commit messages |
| `fixup` | Same as `squash`, but silently discards this commit's message (keeps only the previous commit's message) — the one to use for "oops, typo" commits |
| `drop` | Discard the commit entirely (equivalent to deleting the line) |
| `exec <cmd>` | Run an arbitrary shell command at this point in the rebase (e.g., run tests after each commit) |

Common real workflow: you made 6 messy WIP commits on a feature branch (`wip`, `fix typo`, `actually fix it`, `add tests`, `oops forgot file`, `final`). Before opening a PR, `git rebase -i main`, mark the real logical commits `pick`, mark the fixups as `fixup` against the commit they're fixing, and `reword` the surviving commit into a clean message. The PR reviewer sees one (or a few) coherent commits instead of your actual, embarrassing process.

`--onto` is the advanced form worth knowing: `git rebase --onto newbase oldbase branch` replays only the commits reachable from `branch` but not from `oldbase`, onto `newbase` — used for surgically moving a range of commits (e.g., pulling a feature off a branch it was accidentally built on top of, onto the correct base).

## Cherry-pick: replaying exactly one commit

`git cherry-pick <sha>` takes the diff introduced by a single commit (regardless of what branch it's on) and applies it as a new commit on your current `HEAD`. It's rebase's replay mechanism, scoped to one commit instead of a whole range.

```bash
git cherry-pick abc1234              # apply abc1234's changes as a new commit here
git cherry-pick abc1234 def5678       # apply multiple, in order
git cherry-pick -x abc1234            # append "(cherry picked from commit abc1234)" to the message — do this for traceability
git cherry-pick --no-commit abc1234   # apply the changes to the working dir/index but don't commit yet (lets you fold in more edits first)
git cherry-pick --continue            # after resolving a conflict, same idea as rebase --continue
git cherry-pick --abort               # bail out
```

**When to reach for cherry-pick instead of merge/rebase:**
- **Hotfix backport**: a critical fix lands on `main`, and you need that exact fix — and *only* that fix — on `release-2.3` without pulling in everything else that's happened on `main` since the release branched.
- **Selectively promoting one commit**: someone's feature branch has one commit you need now (a shared utility, a config fix) but the rest of the branch isn't ready to merge.
- **Recovering a specific commit** after a bad reset or from a deleted branch (see file 3) — `git cherry-pick <sha>` onto your current branch is often simpler than trying to reconstruct a whole rebase.

The tradeoff: cherry-picking the same logical change onto multiple branches creates multiple commits with the same diff but different SHAs and (usually) different parents. If that commit is later merged normally between those branches, Git generally handles it fine (it recognizes the diff is already present and produces an empty merge), but relying on cherry-pick as your *primary* integration strategy instead of merge/rebase tends to produce hard-to-follow history — use it for the targeted, occasional case, not routine branch integration.

## Decision guide

| Situation | Use |
|---|---|
| Integrating a finished feature branch into `main`, branch already pushed/shared | **Merge** (non-destructive, safe for shared history) |
| Cleaning up your own local WIP commits before opening a PR | **Interactive rebase** (`git rebase -i`) |
| Keeping your local feature branch up to date with `main` before it's shared | **Rebase** (`git rebase main`), as long as no one else has it |
| Need one specific commit on another branch without the rest of its history | **Cherry-pick** |
| Branch has already been pushed and others may have pulled it | **Merge**, not rebase (golden rule) |

## Points to Remember

- Merge creates a new two-parent commit and never rewrites existing commits — always safe on shared history.
- Rebase replays commits one at a time onto a new base, producing new SHAs for every replayed commit — history is rewritten, not preserved.
- Rebase conflicts are resolved per-commit (`--continue`/`--skip`/`--abort`); merge conflicts are resolved once, for the whole combined diff.
- The golden rule: never rebase commits other people may have already pulled or built on top of — merge instead once history is shared.
- `git rebase -i` verbs: `pick` (keep), `reword` (edit message), `edit` (stop and amend), `squash`/`fixup` (combine into previous, `fixup` discards the message), `drop` (delete).
- Cherry-pick applies one commit's diff as a new commit anywhere — the standard tool for hotfix backports and pulling in a single commit without a full merge/rebase.

## Common Mistakes

- Rebasing (or force-pushing after a rebase of) a branch other people have already pulled, then being confused why their next pull duplicates every commit or conflicts wholesale.
- Treating `squash` and `fixup` as interchangeable — `fixup` silently drops the commit message, `squash` prompts you to combine both messages; picking the wrong one either loses useful context or leaves noisy "fix typo" text in the final message.
- Aborting a rebase with `git reset --hard` instead of `git rebase --abort` — usually works, but `--abort` is the tool actually designed to restore exactly the pre-rebase state and is the safer default.
- Using cherry-pick as the everyday way to move work between long-lived branches instead of merge/rebase — produces duplicate-content commits with different SHAs and history that's hard to reason about later.
- Force-pushing after `rebase -i` without `--force-with-lease`, which can silently overwrite a teammate's commits that were pushed to the same branch after you last fetched.
