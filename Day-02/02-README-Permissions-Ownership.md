# Day 2 — Linux Fundamentals II: Permissions & Ownership

**Phase:** 0 – Foundation | **Week:** W1 | **Domain:** Linux | **Flag:** ⚡ Interview-critical

## Brief

The rwx permission model is the mechanism that turns "who are you" (Day 2, file 1) into "what can you actually do." Nearly every production incident involving "permission denied," a leaked secrets file, or a web server serving the wrong content traces back to a misunderstanding of this model — usually someone reaching for `chmod 777` as a panic button instead of understanding *why* access was denied in the first place. This is consistently one of the highest-yield interview topics in Linux fundamentals because it's easy to test precisely ("what does `chmod 750` mean, and why would a directory need execute permission to be usable at all?").

## Where permissions actually live

Every file and directory has an **inode** — a kernel data structure holding metadata (owner UID, group GID, size, timestamps, and a **mode** field), separate from the file's name (names live in the directory entry, which is why hard links work). The mode field's low 12 bits encode permissions: 3 **special** bits (setuid, setgid, sticky — covered in file 4) followed by 3 groups of 3 bits each for **owner, group, and other** — read (4), write (2), execute (1).

When a process tries to access a file, the kernel checks **in order**: is the process's effective UID the file's owner? If yes, apply the **owner** bits and stop — group/other bits are never consulted as a fallback, even if the owner bits happen to deny access you'd have gotten as "other." Otherwise, is the process's effective GID (or any of its supplementary GIDs) the file's group? If yes, apply **group** bits and stop. Otherwise, apply **other** bits. This "first match wins, no fallback" behavior is a common source of confusion: an owner with `chmod 600` on their own file gets **no** execute even if `other` would have had it, because owner bits are checked exclusively once they match.

## Reading `ls -l` output

```
-rwxr-xr--  1 alice devteam  4096 Jul 12 10:00 deploy.sh
```

The first character is the **file type** (`-` regular file, `d` directory, `l` symlink, `c`/`b` device files). The next nine characters are three rwx triplets: owner (`rwx`), group (`r-x`), other (`r--`). Read top-to-bottom-left-to-right: owner can read/write/execute, group can read/execute, other can only read.

## What rwx means on files vs. directories

This is the part interviewers actually probe, because the meaning **changes** for directories:

| Bit | On a file | On a directory |
|---|---|---|
| `r` | Read file contents | **List** filenames inside (`ls`) — but not their metadata without `x` too |
| `w` | Modify/truncate contents | **Create, delete, or rename entries** inside — this is a directory-level permission, independent of the target file's own permissions |
| `x` | Execute as a program/script | **Traverse** (`cd` into it, or access anything by full path through it) — without `x`, you can't reach anything inside even if you know the exact filename |

The counterintuitive consequence: **you can delete a file you don't own and have no permission to write to, as long as you have write permission on its containing directory** — because deleting an entry is a directory operation, not a file operation (this is exactly what the sticky bit, covered in file 4, exists to prevent on shared directories like `/tmp`). Similarly, a file with full `rwx` for everyone is still completely inaccessible if the directory containing it lacks execute permission for you.

## `chmod` — changing permissions

**Octal (numeric) mode** — each digit is the sum of read(4)+write(2)+execute(1) for owner/group/other respectively:

```bash
chmod 755 script.sh   # rwxr-xr-x — owner full, group+other read/execute (typical for scripts/binaries)
chmod 644 file.txt     # rw-r--r-- — owner read/write, everyone else read-only (typical for regular files)
chmod 700 private/     # rwx------ — owner only, nobody else can even list it
chmod 600 id_rsa        # rw------- — owner read/write only (SSH private keys, secrets)
```

**Symbolic mode** — targets (`u` owner, `g` group, `o` other, `a` all) + operator (`+` add, `-` remove, `=` set exactly) + permission letters:

```bash
chmod u+x script.sh        # add execute for owner only
chmod g-w file.txt          # remove write for group
chmod o=r file.txt           # set "other" to exactly read, nothing else
chmod a+r file.txt            # add read for everyone
chmod -R g+rX /srv/shared/    # recursive; capital X = add execute ONLY on directories or files already executable for someone
```

Capital `X` (vs lowercase `x`) is deliberately different: `chmod -R +x` on a directory tree would make **every** file executable, including config files and data — `+X` only sets execute where it's structurally needed (directories, or files that already have execute for somebody), which is exactly the safe way to fix "can't traverse into subdirectories" without corrupting file semantics.

## `chown` — changing ownership

```bash
chown alice file.txt          # change owner only
chown alice:devteam file.txt  # change owner AND group in one call
chown :devteam file.txt        # change group only (equivalent to chgrp devteam file.txt)
chown -R alice:devteam /srv/app/   # recursive — use with care, changes EVERY file/dir underneath
```

Only **root** can change a file's owner to an arbitrary user on standard Linux configurations (changing ownership away from yourself would otherwise let you dodge disk quotas or disclaim files, so it's a privileged operation). Changing the **group** is looser: the owner can `chgrp`/`chown :group` to any group **they themselves belong to**, without needing root.

## Where ownership comes from on creation

A new file's owner is the **effective UID** of the process that created it (not necessarily the real UID — relevant once you've read the SUID note). Its group is normally the creating process's **effective primary GID** — *unless* the parent directory has the **SGID bit** set, in which case the new file inherits the parent directory's group instead (this is the standard pattern for shared team directories, detailed in file 4).

## `umask` — the default permission mask

New files/directories don't get `777`/`666` by default — they get `777`/`666` **minus** the umask. The kernel/shell subtracts (bitwise) the umask value from the maximum:

```bash
umask        # show current mask, e.g. 0022
```

With `umask 022`: new directories become `755` (`777 - 022`), new files become `644` (`666 - 022` — files never get a default execute bit from creation, only directories and explicit `chmod +x` do). Setting `umask 077` in a script or shell profile is a common hardening step to ensure anything created is private-by-default (`700`/`600`) unless explicitly opened up afterward.

## Points to Remember

- Permission checks are **owner OR group OR other**, in that priority order, with no fallback — matching the owner category locks in the owner bits even if group/other would've granted more.
- Directory `x` = traverse, directory `w` = create/delete/rename entries inside (independent of the file's own permissions) — this is why a read-only file can still be deletable.
- `chmod -R +X` (capital X) is the safe way to add execute recursively without making plain data files executable.
- Only root can `chown` a file to a different **user**; changing **group** just requires membership in the target group.
- `umask` is subtracted from `777`/`666` — `umask 022` → `755` dirs / `644` files; files never get an execute bit purely from creation.

## Common Mistakes

- Reaching for `chmod 777` (or `chmod -R 777`) as a generic fix for "permission denied" instead of diagnosing which specific bit (often directory execute, not file permissions at all) is actually missing.
- Running `chmod -R +x` on a whole tree to fix a traversal problem, accidentally making every data file "executable" and breaking tools that check for that (and creating a genuine security smell).
- Expecting group permissions to apply to the file's *owner* — they never do; the owner is always evaluated against the owner bits only, never group/other.
- Forgetting that deleting a file depends on the **directory's** write permission, not the file's own permissions — leading to confusion when `rm` succeeds on a `chmod 000` file, or fails on a `777` file sitting in a directory you can't write to.
- Recursively `chown`-ing an entire directory tree (`chown -R`) without checking for symlinks pointing outside the tree, unintentionally changing ownership of files elsewhere via `-R`'s symlink traversal behavior on some tools/flags.
