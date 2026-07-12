# Day 8 — Lab: Git Deep Dive I

**Goal:** Stop treating Git as a black box. Read objects directly with plumbing commands, practice interactive rebase until the verbs are muscle memory, then run the core assigned scenario for today — corrupt `main` with a bad merge, recover it with the reflog, and re-integrate a feature branch cleanly with rebase instead of repeating the mistake.

**Prerequisites:** Git installed (`git --version` — anything 2.20+ is fine). All labs run inside a **throwaway repo in `/tmp`**, so nothing here touches any real project. Set a local identity once if you haven't globally:

```bash
git config --global user.email "you@example.com"
git config --global user.name "Your Name"
```

---

### Lab 1 — Explore Git internals with plumbing commands

1. Create a fresh repo and make one commit:
   ```bash
   mkdir -p /tmp/git-lab && cd /tmp/git-lab
   git init -b main
   echo "hello" > file.txt
   git add file.txt
   git commit -m "first commit"
   ```
2. Walk the object graph by hand, commit -> tree -> blob:
   ```bash
   git cat-file -t HEAD              # commit
   git cat-file -p HEAD              # shows: tree <sha>, author, committer, message

   TREE_SHA=$(git cat-file -p HEAD | awk '/^tree/{print $2}')
   git cat-file -t "$TREE_SHA"       # tree
   git cat-file -p "$TREE_SHA"       # 100644 blob <sha>  file.txt

   BLOB_SHA=$(git cat-file -p "$TREE_SHA" | awk '{print $3}')
   git cat-file -t "$BLOB_SHA"       # blob
   git cat-file -p "$BLOB_SHA"       # hello
   ```
3. Prove hashing is pure content-addressing — compute the blob's hash without ever touching the repo:
   ```bash
   echo "hello" | git hash-object --stdin
   ```
   Confirm it matches `$BLOB_SHA` exactly, character for character.
4. Inspect the ref that makes this all navigable:
   ```bash
   cat .git/HEAD                     # ref: refs/heads/main
   cat .git/refs/heads/main            # the commit sha, as plain text
   git rev-parse HEAD                  # resolves the same sha via the porcelain command
   git ls-tree HEAD                    # same tree listing, via the ls-tree plumbing command
   ```

**Success criteria:** You can chase commit -> tree -> blob by hand using `cat-file`, and you can explain in one sentence why `echo "hello" | git hash-object --stdin` produces the exact same SHA as the blob already stored in the repo.

---

### Lab 2 — Interactive rebase practice

1. In the same repo, build up some intentionally messy history:
   ```bash
   echo "second"  >> file.txt && git commit -am "wip: second change"
   echo "third"   >> file.txt && git commit -am "typo fix"
   echo "fourth"  >> file.txt && git commit -am "add real feature"
   echo "debug"   >> file.txt && git commit -am "oops debug print"
   git log --oneline
   ```
2. Clean it up with interactive rebase:
   ```bash
   git rebase -i HEAD~4
   ```
   In the editor that opens, edit the plan so that:
   - `wip: second change` stays `pick`
   - `typo fix` becomes `fixup` (folds into the commit above, message discarded)
   - `add real feature` stays `pick`
   - `oops debug print` becomes `drop` (discarded entirely)
3. Save and close the editor. Confirm the result:
   ```bash
   git log --oneline
   cat file.txt         # should contain hello/second/third/fourth — NOT "debug"
   ```
4. Now practice `reword`: run `git rebase -i HEAD~2`, mark the last commit `reword`, and change its message when prompted. Confirm with `git log --oneline`.
5. (Optional, for scripting/CI use) The whole interactive flow can be driven non-interactively with `GIT_SEQUENCE_EDITOR`, which is worth knowing exists even though you should do this by hand at least once:
   ```bash
   GIT_SEQUENCE_EDITOR="sed -i '' -e '2s/pick/fixup/' -e '4s/pick/drop/'" git rebase -i HEAD~4
   ```

**Success criteria:** You can predict, before running `git log --oneline`, exactly what the resulting history will look like from a rebase todo list — and you know the difference between `squash` and `fixup` without checking notes.

---

### Lab 3 — Core activity: corrupt `main`, recover with reflog, rebase cleanly

This is the assigned hands-on activity for today. Do it in a fresh repo so the history is unambiguous.

1. **Set up a realistic diverging history:**
   ```bash
   mkdir -p /tmp/git-lab-2 && cd /tmp/git-lab-2
   git init -b main

   echo "v1" > app.txt && git add app.txt && git commit -m "1: initial app"
   echo "v2" >> app.txt && git commit -am "2: add v2"
   echo "v3" >> app.txt && git commit -am "3: add v3"

   git checkout -b feature
   echo "feature-config" > feature.txt && git add feature.txt && git commit -m "4: add feature.txt"
   echo "v3-feature-tweak" >> app.txt && git commit -am "5: feature tweaks app.txt"

   git checkout main
   echo "v3-hotfix" >> app.txt && git commit -am "6: hotfix on main"
   ```
2. Look at the divergence before touching anything:
   ```bash
   git log --graph --oneline --decorate --all
   ```
   `main` and `feature` both edited the last line of `app.txt` independently since they split — this **will** conflict on merge.
3. **Simulate a bad merge that corrupts `main`.** Attempt the merge, hit the conflict, then resolve it carelessly by blindly taking `feature`'s side of everything — a very real mistake under deadline pressure:
   ```bash
   git merge feature
   # CONFLICT (content): Merge conflict in app.txt
   git checkout --theirs app.txt
   git add app.txt feature.txt
   git commit -m "7: merge feature into main (BAD merge resolution)"
   ```
4. **Discover the corruption** — the hotfix from commit 6 is now silently gone from `main`:
   ```bash
   cat app.txt              # "v3-hotfix" line is MISSING — main is corrupted
   grep -c "v3-hotfix" app.txt || echo "CONFIRMED: hotfix lost"
   ```
5. **Recover using the reflog.** Find the state of `main` immediately before the bad merge commit, and reset back to it:
   ```bash
   git reflog
   # look for the entry just above "commit (merge): 7: merge feature into main..." —
   # that's "commit: 6: hotfix on main"
   git reset --hard HEAD@{1}
   cat app.txt               # "v3-hotfix" line is back — main is restored
   ```
6. **Re-integrate `feature` properly, via rebase instead of merge this time**, resolving the real conflict correctly instead of blindly taking one side:
   ```bash
   git checkout feature
   git rebase main
   # CONFLICT (content): Merge conflict in app.txt on "5: feature tweaks app.txt"
   ```
   Open `app.txt`, resolve the conflict by **keeping both lines** (the hotfix and the feature tweak), then:
   ```bash
   git add app.txt
   git rebase --continue
   ```
7. Fast-forward `main` onto the now-clean, rebased `feature`, and confirm the final graph is linear with nothing lost:
   ```bash
   git checkout main
   git merge feature          # Fast-forward
   git log --graph --oneline --decorate --all
   cat app.txt                 # v1 / v2 / v3 / v3-hotfix / v3-feature-tweak — everything present
   cat feature.txt              # feature-config — also present
   ```

**Success criteria:** You can explain, in your own words, why `HEAD@{1}` in step 5 pointed at the pre-merge commit and not just "one commit back" (`HEAD~1`) — and you ended up with a linear history containing the hotfix, the feature.txt addition, and a correctly merged app.txt, with zero content lost.

---

### Lab 4 — Cherry-pick: hotfix backport

1. Set up a repo simulating a tagged release with ongoing main-branch work:
   ```bash
   mkdir -p /tmp/git-lab-3 && cd /tmp/git-lab-3
   git init -b main

   echo "v1.0" > VERSION && git add VERSION && git commit -m "1: release v1.0"
   git branch release-1.0

   echo "new feature work" > feature2.txt && git add feature2.txt && git commit -m "2: unrelated feature work on main"
   echo "SECURITY_PATCH=true" >> VERSION && git commit -am "3: critical security hotfix"
   ```
2. `release-1.0` needs the security fix from commit 3, but **not** the unrelated feature work from commit 2. Cherry-pick just the hotfix:
   ```bash
   HOTFIX_SHA=$(git rev-parse main)
   git checkout release-1.0
   git cherry-pick -x "$HOTFIX_SHA"
   ```
3. Confirm the result:
   ```bash
   git log --oneline
   cat VERSION                  # has SECURITY_PATCH=true
   ls                            # feature2.txt is NOT here — it never came along
   git log -1 --format="%B"     # message includes "(cherry picked from commit ...)"
   ```
4. Now practice a conflicting cherry-pick: on `main`, make one more commit that changes the same `VERSION` line the hotfix touched, then try cherry-picking that new commit onto `release-1.0` too. Resolve the conflict, `git add`, and `git cherry-pick --continue`.

**Success criteria:** You can explain why `-x` matters (it records the source commit's SHA in the message, which is how you or a teammate later trace a backported fix to its origin), and you've resolved at least one real cherry-pick conflict end to end.

---

### Cleanup

```bash
rm -rf /tmp/git-lab /tmp/git-lab-2 /tmp/git-lab-3
```

### Stretch challenge

Simulate the worst-case version of "I deleted the wrong branch": create a branch, check it out, commit something important on it, switch back to `main`, then delete it with `git branch -D` (not just reset).

```bash
git checkout -b throwaway
echo "important work" > important.txt
git add important.txt && git commit -m "important work nobody should lose"
git checkout main
git branch -D throwaway
```

First, recover it using `git reflog` (look for the `checkout: moving from throwaway to main` entry — the commit just before it is the branch's last tip) and re-create the branch with `git branch recovered-throwaway <sha>`.

Then prove you don't strictly need the reflog either: run `git fsck --full --unreachable --no-reflogs` (the `--no-reflogs` flag makes fsck ignore reflog entries, simulating what happens once they've expired) and confirm it still reports the commit, tree, and blob as `unreachable`. Recover the same way with `git branch recovered-throwaway-2 <sha>` and confirm `important.txt` comes back. Write one sentence explaining the actual difference between the reflog path and the fsck path to recovery.
