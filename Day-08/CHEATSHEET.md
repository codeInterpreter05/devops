# Day 8 — Cheatsheet: Git Deep Dive I

## Object model / plumbing

```bash
git cat-file -t <sha>                  # show object type: blob, tree, commit, tag
git cat-file -p <sha>                  # pretty-print object contents (works for any type)
git cat-file -s <sha>                  # show object size in bytes

git hash-object <file>                  # compute the sha a file WOULD get, don't store it
git hash-object -w <file>               # compute AND write it into .git/objects
echo "text" | git hash-object --stdin    # same, from stdin

git ls-tree <sha>                       # list a tree's direct entries (mode, type, sha, name)
git ls-tree -r <sha>                    # recursive — include blobs in subdirectories

git rev-parse HEAD                      # resolve a ref/shorthand to its full sha
git rev-parse --short HEAD               # abbreviated sha
git rev-parse HEAD^                      # first parent of HEAD
git rev-parse HEAD~3                     # 3 commits back in ancestry (first-parent chain)

git write-tree                          # write the current index as a tree object
git commit-tree <tree-sha> -p <parent-sha> -m "msg"   # build a commit object by hand
git update-ref refs/heads/main <sha>      # move a branch ref to point at a specific commit
```

## `.git` directory map

```
.git/HEAD              # symbolic ref: which branch you're on ("ref: refs/heads/main")
.git/config             # repo-local config
.git/index               # staging area (binary)
.git/objects/            # every blob/tree/commit/tag — content-addressed by sha
.git/refs/heads/          # local branches (one file per branch, or packed-refs)
.git/refs/tags/            # tags
.git/logs/HEAD             # HEAD's reflog
.git/logs/refs/heads/<b>    # per-branch reflog
```

## `git log` / graph

```bash
git log --oneline                            # one line per commit: <short-sha> <subject>
git log --graph --oneline --decorate --all    # full DAG, all refs, branch/tag labels
git log --graph --oneline --decorate --all --simplify-by-decoration   # collapse to just branch/tag points
git log -p                                    # show the diff introduced by each commit
git log --stat                                 # show files changed + insertion/deletion counts per commit
git log <branch1>..<branch2>                    # commits in branch2 not in branch1
git log --author="name"                          # filter by author
git show <sha>                                    # full diff + metadata for one commit
```

## Interactive rebase

```bash
git rebase -i HEAD~N          # edit the last N commits on the current branch
git rebase -i main             # edit everything since branch diverged from main
git rebase --continue           # after resolving a conflict / editing during `edit`
git rebase --skip                # discard the current commit entirely, move to next
git rebase --abort                # cancel, restore branch to pre-rebase state exactly

git rebase --onto <newbase> <oldbase> <branch>   # replay only branch's commits not in oldbase, onto newbase
git rebase -i --autosquash HEAD~N                  # auto-order fixup!/squash! commits (see below)
git commit --fixup=<sha>                             # create a fixup commit targeting <sha>
git commit --squash=<sha>                             # create a squash commit targeting <sha>
```

Rebase todo verbs (edit the list, don't type these as flags):

```
pick    keep commit as-is
reword   keep changes, stop to edit the commit message
edit      stop right after applying this commit, to amend it
squash   combine into previous commit, merge both messages
fixup     combine into previous commit, DISCARD this message
drop      remove the commit entirely
exec <cmd>   run a shell command at this point in the rebase
```

## Cherry-pick

```bash
git cherry-pick <sha>                 # apply one commit's diff as a new commit on HEAD
git cherry-pick <sha1> <sha2>           # apply multiple, in order given
git cherry-pick -x <sha>                 # append "(cherry picked from commit <sha>)" to message
git cherry-pick --no-commit <sha>         # apply changes to working dir/index, don't commit yet
git cherry-pick --continue                 # after resolving a conflict
git cherry-pick --skip                      # skip this commit, continue with the rest
git cherry-pick --abort                      # cancel, restore pre-cherry-pick state
```

## Reflog & recovery

```bash
git reflog                        # = git reflog show HEAD — chronological log of HEAD movements
git reflog show <branch>           # per-branch reflog
git reflog expire --expire=now --all   # manually expire all reflog entries (careful)

HEAD@{0}                          # where HEAD is right now (reflog-relative)
HEAD@{2}                           # where HEAD was 2 movements ago (reflog-relative)
HEAD~2                              # 2 commits back in ANCESTRY (different from HEAD@{2}!)
main@{yesterday}                     # where `main` pointed at a given time

git reset --hard HEAD@{2}          # move current branch back to a past reflog position
git branch recovered <sha>          # recreate a branch pointer at a specific (possibly dangling) sha, non-destructively
git checkout <sha>                   # inspect a commit directly (detached HEAD) before deciding
```

## `fsck` / `gc` / `prune`

```bash
git fsck --full --unreachable           # list every object not reachable from any ref/reflog
git fsck --full --unreachable --no-reflogs   # same, but ignore reflog entries (simulates expiry)
git fsck --lost-found                     # dump dangling commits/blobs into .git/lost-found/{commit,other}

git gc                                     # normal housekeeping, respects reflog expiry windows
git gc --prune=now                          # aggressively prune ALL unreachable objects, no grace period
git prune --expire=now                       # prune loose unreachable objects immediately

git config gc.reflogExpire            # default: 90 days (entries for still-reachable commits)
git config gc.reflogExpireUnreachable   # default: 30 days (entries for unreachable commits)
```

## Quick decision reference

```
merge          non-destructive, safe on shared/pushed branches, adds a merge commit
rebase          rewrites history (new SHAs), only on local/unshared branches
cherry-pick      one specific commit onto another branch (hotfix backport)
reflog           recover from bad reset/rebase/checkout, LOCAL ONLY, has an expiry
fsck --lost-found  recover when reflog doesn't have it (deleted branch never checked out, or expired)
```
