# Day 1 — Linux Fundamentals I: Essential Commands

**Phase:** 0 – Foundation | **Week:** W1 | **Domain:** Linux | **Flag:** ⚡ Interview-critical

## Brief

These six commands (`ls`, `cd`, `cp`, `mv`, `rm`, `find`) are the ones you'll type more than any others for the rest of your career. Knowing their common flags cold — not just "I know what `ls` does" but "I know `ls -lart` and why I'd type it in that order" — is a fast, reliable signal of hands-on experience.

## `ls` — list directory contents

```bash
ls -la          # long format, include hidden (dotfiles)
ls -lh          # long format, human-readable sizes (K/M/G instead of bytes)
ls -lart        # long, all, reverse, sorted by time (oldest last, newest at the bottom — good for scrolling logs of files)
ls -d */        # list only directories
```
Hidden files (dotfiles like `.bashrc`, `.ssh`) are just files whose name starts with `.` — there's no special "hidden" attribute like on Windows. `ls` simply filters them out by default.

## `cd` — change directory

```bash
cd /var/log      # absolute
cd ../..         # up two levels, relative
cd -             # jump back to the PREVIOUS directory (toggles)
cd               # no args -> go home (~)
```
`cd -` is underused and extremely handy when bouncing between two directories repeatedly.

## `cp` — copy files/directories

```bash
cp file.txt file.bak          # copy a file
cp -r sourcedir/ destdir/     # recursive, required for directories
cp -a sourcedir/ destdir/     # archive mode: preserves permissions, timestamps, symlinks — use for backups
cp -v                         # verbose — show what's happening
cp -i                         # interactive — prompt before overwrite (good habit to alias cp to this)
```
`-a` (archive) is what you want for anything backup-related; plain `cp -r` can silently change permissions/ownership on the copy.

## `mv` — move / rename

```bash
mv old.txt new.txt        # rename (same directory = rename, different directory = move)
mv file.txt /tmp/         # move into a directory
mv -i file.txt dest/      # prompt before overwrite
```
There is no separate "rename" command — `mv` does both, based on whether the destination is in the same directory.

## `rm` — remove files/directories

```bash
rm file.txt          # delete a file
rm -r dir/           # recursive delete of a directory
rm -rf dir/          # recursive + force (no prompts, ignores missing files) — THE dangerous one
rm -i file.txt        # prompt before each deletion
```
**There is no undo.** No trash bin, no recycle bin, by default. Before any `rm -rf`, get in the habit of running the equivalent `ls` or `find` first to see exactly what will be deleted, and always quote variables: `rm -rf -- "$TARGET_DIR"` (the `--` stops a `$TARGET_DIR` that happens to start with `-` from being parsed as a flag).

## `find` — search the filesystem by walking it live

```bash
find /etc -name "*.conf"              # find files by exact name pattern (case-sensitive)
find /etc -iname "*.conf"             # case-insensitive
find / -maxdepth 2 -type d            # limit recursion depth; -type d = directories only, -type f = files only
find /var/log -mtime -1               # modified in the last 1 day
find / -size +100M                    # files larger than 100MB
find /tmp -mtime +7 -delete           # find files older than 7 days and delete them — powerful, be careful
find . -name "*.log" | xargs grep -l "ERROR"   # combine find + xargs + grep to search inside matched files
```
`find` walks the actual filesystem tree at the time you run it (always current, can be slow on huge trees). `locate` (if installed) queries a pre-built database (`updatedb`) — much faster, but can be stale if files changed since the last index update. For quick "does this exist somewhere" queries, `locate`; for precise, current-state, condition-based searches (by time, size, permission, and combined with `-exec`), `find`.

## Points to Remember

- `cp -a` for anything backup-related (preserves metadata); plain `cp -r` does not guarantee that.
- `rm` has no undo — always preview with `find`/`ls` before `rm -rf`, and quote path variables.
- `mv` = rename when destination is same directory, move when it's not — there's no separate rename command.
- `find` walks the tree live (always current, slower); `locate` queries a stale-but-fast prebuilt index.
- `find ... -exec cmd {} \;` (or `-exec cmd {} +` for batching) lets you act on every match without piping through `xargs`.

## Common Mistakes

- Running `rm -rf $DIR/*` where `$DIR` is empty/unset — expands to `rm -rf /*` (root-level catastrophe). Always quote and validate: `rm -rf -- "${DIR:?}/"*`.
- Forgetting `-r` for `cp`/`rm` on directories and getting a confusing "is a directory" error, then reflexively adding `-rf` everywhere out of habit (dangerous muscle memory).
- Using `find / -name X` without `-maxdepth` or excluding `/proc`, causing the search to crawl huge virtual filesystems needlessly (slow, sometimes throws permission-denied noise for every protected directory).
- Piping `find` results into `grep` for filename matching (`find . | grep pattern`) instead of `find . -name "pattern"` — works, but breaks on filenames with newlines/spaces and is slower.
- Not knowing `cd -` exists, and instead re-typing full paths to bounce between two directories.
