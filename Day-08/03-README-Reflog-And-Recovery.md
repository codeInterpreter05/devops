# Day 8 — Git Deep Dive I: Reflog & Recovery

**Phase:** 0 – Foundation | **Week:** W2 | **Domain:** Git | **Flag:** ⚡ Interview-critical

## Brief

Every engineer eventually does something that looks like it destroyed work: a `git reset --hard` to the wrong commit, an interactive rebase that dropped the wrong line, a `git branch -D` on a branch they meant to keep. The instinctive reaction is panic; the correct reaction is `git reflog`. Understanding what the reflog actually is — and just as importantly, what it *isn't* — is what separates "I recovered it in ten seconds" from either needless panic over recoverable mistakes, or false confidence that everything is always recoverable (it isn't).

## What the reflog actually tracks

The reflog is a **local, per-repository log of every point `HEAD` (and each branch ref) has pointed to**, in order, with a description of the operation that moved it. It is *not* a history feature in the sense that `git log` is — `git log` follows the commit graph's parent pointers (what's reachable from a ref *right now*); the reflog is a flat, chronological log of **ref movements on this one clone**, regardless of whether those commits are still reachable any other way.

```bash
git reflog                     # equivalent to: git reflog show HEAD
git reflog show main            # per-branch reflog, not just HEAD's
```

```
a1b2c3d HEAD@{0}: rebase (finish): returning to refs/heads/feature
e4f5g6h HEAD@{1}: rebase (pick): Add validation
h7i8j9k HEAD@{2}: rebase (start): checkout main
q6r7s8t HEAD@{3}: reset: moving to HEAD~3
n3o4p5q HEAD@{4}: commit: Add remember-me checkbox
```

Every `checkout`, `commit`, `merge`, `rebase`, `reset`, `cherry-pick`, and `pull` writes an entry. This is stored per-ref, in plain text, at `.git/logs/HEAD` and `.git/logs/refs/heads/<branch>` — read them directly if you want, they're not a database, just append-only log files.

Three properties matter enormously in practice:

1. **It's local only.** The reflog lives in your `.git` directory and is never pushed, fetched, or cloned. A teammate's clone has no knowledge of *your* reflog, and cloning a repo fresh gives you an empty one. Reflog recovery only works on the machine/clone where the mistake happened.
2. **It has an expiry.** Entries aren't kept forever. By default, reflog entries for commits still reachable from some ref expire after 90 days (`gc.reflogExpire`), and entries for commits that are *unreachable* from anything expire after 30 days (`gc.reflogExpireUnreachable`) — both configurable, and both only actually pruned when `git gc` runs (which Git does automatically and periodically, including sometimes as a side effect of routine commands).
3. **It tracks ref movements, not object existence.** The reflog is a convenient index into "what did this ref used to point at," which lets you *find* a commit's SHA again. The commit object itself is recoverable as long as it (and everything it needs — its tree, all blobs) hasn't actually been deleted from `.git/objects`. The reflog is the map; the objects themselves are the territory.

## Recovering from a bad reset, rebase, or checkout

The general recipe is always the same: **find the SHA you want in the reflog, then point a ref at it.**

```bash
git reflog                              # find the entry from just before the mistake
git reset --hard HEAD@{2}                 # move the current branch back to that point
# or, non-destructively, without touching your current branch:
git branch recovered-work HEAD@{2}        # create a new branch pointing at the old state
git checkout recovered-work                # inspect it before deciding what to do next
```

`HEAD@{n}` is a **reflog-relative reference** — it means "wherever `HEAD` was `n` movements ago," which is different from `HEAD~n` (which means "`n` commits back in the current commit's ancestry"). This distinction matters constantly during recovery: after a rebase or reset, the commit ancestry has changed, but the reflog still remembers exactly where you were before.

This same recipe handles the common disaster scenarios:

- **Bad `git reset --hard`**: the previous commit is still in the reflog (`reset: moving to ...` entries show exactly where you came from). Reset back to the entry right before it.
- **Bad interactive rebase** (dropped/squashed the wrong commit): `rebase (start)` in the reflog marks where the branch was before the rebase began — reset the branch back to that entry, and you're exactly where you started, before ever running `rebase -i`.
- **Deleted a branch** (`git branch -D feature`): the branch's own reflog file is deleted along with it, but if you had that branch checked out at any point, `HEAD`'s reflog still has entries recording its tip SHA. `git reflog` (on `HEAD`, not the now-gone branch) will show `checkout: moving from feature to main`-style entries — the SHA just before that is the branch's last tip. Recreate it: `git branch feature <that-sha>`.

## `git fsck` when the reflog isn't enough

Sometimes the reflog doesn't have what you need — the branch was deleted without ever being checked out on this clone, or the relevant reflog entries already expired. The next line of defense is `git fsck`, which walks every object in `.git/objects` and reports ones that aren't reachable from any ref:

```bash
git fsck --full --unreachable          # list every object not reachable from any ref/reflog entry
git fsck --lost-found                   # write dangling commits/blobs into .git/lost-found/{commit,other}
git cat-file -p <sha>                   # inspect any object fsck surfaces, exactly as in file 1
```

`--lost-found` is the closest thing Git has to an actual "Recycle Bin": it dumps every dangling (unreferenced but still present) commit into `.git/lost-found/commit/` and every other dangling object into `.git/lost-found/other/`, so you can `cat-file -p` each one, find the commit message/content you're looking for, and re-attach it with `git branch recovered <sha>`.

This works precisely because deleting a ref (a branch) or rewriting history (a rebase) **does not delete the underlying objects** — it only removes the pointer to them. The commit, its tree, and its blobs sit in `.git/objects` completely intact, just unreferenced, until something explicitly garbage-collects them.

## Reflog recovery vs. true data loss

This is the distinction that actually matters, and the one worth being precise about in an interview: **"deleted" in Git almost always means "unreferenced," not "gone."** An object becomes truly, unrecoverably gone only when:

1. It's unreachable from every ref, remote-tracking ref, stash, *and* every reflog entry (i.e., its expiry window has passed, or `--expire=now` was used), **and**
2. Something actually runs garbage collection — `git gc`, or explicitly `git prune` / `git gc --prune=now`.

```bash
git gc                    # normal housekeeping — repacks objects, respects reflog expiry windows
git gc --prune=now         # aggressively prune ALL unreachable objects immediately, ignoring the grace period
git prune --expire=now     # prune loose unreachable objects right now, no grace period
```

Until that gc/prune actually runs, "lost" commits are sitting untouched in `.git/objects`, findable via reflog or `fsck`. This is why the standard advice after any git disaster is "stop running more git commands, don't run `git gc`, and go look at the reflog/fsck first" — the danger isn't the mistake itself, it's compounding it by forcing a garbage collection before you've recovered what you need. In practice, automatic gc triggers on its own after enough loose objects accumulate (`git gc --auto`, run implicitly by many porcelain commands) — which is exactly why reflog/fsck recovery is a "usually works, do it promptly" tool, not an infinite safety net.

## Points to Remember

- The reflog is a local, per-ref, chronological log of ref movements (`HEAD` and each branch) — not pushed, not cloned, not a substitute for `git log`.
- Reflog entries expire (90 days reachable / 30 days unreachable by default) and are only actually removed when `git gc` runs — until then, "deleted" commits are just unreferenced, not gone.
- `HEAD@{n}` means "where HEAD was n movements ago" (reflog-relative); `HEAD~n` means "n commits back in ancestry" — they diverge after any reset/rebase/checkout.
- `git reflog` finds SHAs by ref history; `git fsck --unreachable` / `--lost-found` finds dangling objects even when no reflog entry points at them (e.g., a branch deleted without ever being checked out here).
- True, unrecoverable data loss requires both "unreachable from every ref and every reflog entry" AND an actual `git gc`/`prune` run — the two together, not either alone.
- Recover non-destructively when unsure: `git branch recovered-work <sha>` to inspect first, rather than `git reset --hard` straight onto your current branch.

## Common Mistakes

- Panicking and running `git gc` "to clean things up" immediately after a mistake — this is the one thing that can turn a recoverable ref problem into permanent data loss if the objects were already past their expiry window.
- Confusing `HEAD@{2}` (reflog position) with `HEAD~2` (ancestry position) — they frequently point at completely different commits once a reset or rebase has happened.
- Assuming the reflog is shared or backed up — it's local to one `.git` directory; a fresh clone or a teammate's machine has none of your reflog history.
- Forgetting that a deleted branch's *own* reflog dies with it — recovery in that case relies on `HEAD`'s reflog (if you ever checked the branch out) or `git fsck`, not `git reflog show <deleted-branch>` (which will just error).
- Treating reflog recovery as permanent safety net with no time limit — it has a real, configurable expiry, and relying on very old reflog entries without checking `gc.reflogExpire` is a bad bet.
