# Day 2 — Linux Fundamentals II: Special Permission Bits (SUID, SGID, Sticky)

**Phase:** 0 – Foundation | **Week:** W1 | **Domain:** Linux | **Flag:** ⚡ Interview-critical

## Brief

Beyond the standard rwx triplets sit three **special bits** — SUID, SGID, and sticky — that solve three narrow but important problems: letting an ordinary user run a specific program with elevated privilege, making shared directories automatically use the right group, and stopping people from deleting each other's files in a world-writable directory. These are small mechanisms with outsized interview weight, because **"what's the difference between SUID and the sticky bit?" is one of the most common Linux interview questions in existence** — precisely because a correct answer proves you understand effective UID/GID, not just permission syntax.

## SUID — Set User ID (octal `4000`)

When SUID is set on an **executable**, the kernel runs the resulting process with its **effective UID set to the file's owner**, not the UID of whoever launched it. The **real UID** (who actually invoked it) doesn't change — only the **effective UID** (whose permissions get checked from then on) does. This is exactly how an unprivileged user changes their own password:

```bash
ls -l /usr/bin/passwd
# -rwsr-xr-x 1 root root ... /usr/bin/passwd
```

The lowercase `s` in the owner-execute slot means "SUID set **and** executable." `/etc/shadow` is root-only writable, yet any user can run `passwd` to update their own hashed entry — because while `passwd` is running, its effective UID is `root` (the file's owner), regardless of who launched it. Once the program exits, that elevation disappears with it — it's scoped to the lifetime of that one process, not a persistent privilege grant.

**Important gotchas:**
- SUID has **no effect on directories** — it's an execution-time behavior, meaningless without something being executed.
- Modern Linux kernels **ignore the SUID bit on scripts** (anything starting with a shebang, `#!/bin/bash` etc.) for security reasons — historically, a race condition between the kernel opening the script and the interpreter re-opening it could be exploited to run arbitrary code as the file owner (a classic TOCTOU bug). SUID only reliably works on **compiled binaries**. This surprises people who `chmod 4755` a shell script expecting it to behave like a SUID C program.
- Every SUID-root binary on a system is a potential privilege-escalation target if it has a bug (buggy input handling, ability to spawn a shell, etc.) — auditing for **unexpected** SUID binaries is a standard security hygiene practice:
  ```bash
  find / -perm -4000 -type f 2>/dev/null    # every SUID binary on the system
  ```
  A stray SUID binary that shouldn't be there (leftover from a build, an unnoticed package default) is a real, common finding in security audits and CTF-style privilege-escalation exercises.

## SGID — Set Group ID (octal `2000`)

SGID behaves like SUID but for **groups**, and has a second, very different behavior on **directories**:

- **On an executable**: the process's effective GID becomes the file's **group**, not the invoking user's primary group.
- **On a directory**: any file or subdirectory created inside **inherits the directory's group** (instead of the creating user's own primary group) — and, on most Linux filesystems, new subdirectories also inherit the SGID bit itself, so the behavior propagates down the tree automatically.

```bash
mkdir /srv/devteam
chown root:devteam /srv/devteam
chmod 2775 /srv/devteam          # rwxrwsr-x — SGID set, group has rwx
ls -ld /srv/devteam
# drwxrwsr-x 2 root devteam ... /srv/devteam
```

This is the standard pattern for a **shared team directory**: without SGID, if `alice` (primary group `alice`) creates a file inside `/srv/devteam`, that file's group becomes `alice`, and `bob` — even though he's a member of `devteam` — can't necessarily read/write it depending on `other` permissions. With SGID set, every file anyone creates in `/srv/devteam` automatically belongs to group `devteam`, so group permissions apply consistently to the whole team regardless of who authored which file.

## Sticky bit (octal `1000`)

Historically, setting the sticky bit on an **executable** told the kernel to keep its text segment resident in swap for faster reloading — this use is obsolete on modern Linux and has no effect on executables today. Its living, relevant use is entirely on **directories**: it restricts deletion/renaming.

In a normal world-writable directory (`777`), directory write permission lets **anyone** delete or rename **any** file inside — regardless of who owns the individual file — because deletion is governed by directory permissions, not file permissions (see the ownership note in file 2). The sticky bit adds an extra rule on top: **only the file's owner, the directory's owner, or root can delete or rename a file within a sticky directory**, even though the directory is still writable by everyone for *creating* new files.

```bash
ls -ld /tmp
# drwxrwxrwt 12 root root ... /tmp
```

The trailing `t` (lowercase = sticky bit set **and** other-execute already present; uppercase `T` = sticky set but other-execute is **not** set, an unusual combination) is exactly why `/tmp` is safe to share among every user on a multi-user system: everyone can create their own temp files, but nobody can delete or rename files another user created there, even though the directory permissions alone (`777`) would otherwise allow it.

## Symbolic representation summary

| Position | Bit unset | Bit set + underlying x set | Bit set + underlying x NOT set |
|---|---|---|---|
| Owner execute | `x` or `-` | `s` (SUID) | `S` (SUID, but not executable — usually a misconfiguration) |
| Group execute | `x` or `-` | `s` (SGID) | `S` (SGID, but not executable) |
| Other execute | `x` or `-` | `t` (sticky) | `T` (sticky, but not executable) |

## Setting special bits

```bash
chmod 4755 /usr/local/bin/tool     # SUID + rwxr-xr-x
chmod 2775 /srv/devteam            # SGID + rwxrwxr-x
chmod 1777 /tmp                     # sticky + rwxrwxrwx
chmod u+s file                       # symbolic: add SUID
chmod g+s dir                          # symbolic: add SGID
chmod +t dir                            # symbolic: add sticky
```

Using a **4-digit** numeric mode (`4755`) sets/preserves the special bit alongside the standard bits; using a plain **3-digit** mode (`755`) on a file that already had a special bit **clears** it — a common way special bits silently disappear after a routine `chmod 755` "just to fix permissions."

## Answering the interview question directly

**"What is the difference between SUID and the sticky bit? When would you use each?"**

They solve two unrelated problems. **SUID** changes the **effective UID of a running process** to the file's owner — it's about controlled **privilege elevation during execution**, used on trusted executables so an unprivileged user can perform one specific privileged action (the canonical example: `passwd`, letting any user update their own entry in root-only `/etc/shadow`). The **sticky bit** has nothing to do with execution or privilege elevation at all — it's applied to **shared, world-writable directories** to restrict **deletion/renaming** so that only a file's owner (or root) can remove it, even though everyone can write into the directory (the canonical example: `/tmp`, `1777`). Use SUID when you need to grant a narrow, specific privileged capability through a trusted binary; use the sticky bit when you need a directory writable by many users without letting them destroy each other's files.

## Points to Remember

- SUID/SGID affect **effective UID/GID during execution**; sticky bit affects **who can delete/rename inside a directory** — entirely different mechanisms bundled under "special bits" for historical/bit-layout reasons only.
- SUID is ignored on scripts by the kernel (shebang-based files) — it only reliably elevates privilege on compiled binaries.
- SGID on a directory makes new files inherit the directory's **group** (and usually propagates SGID to new subdirectories) — the standard fix for shared team directories.
- A 3-digit `chmod` on a file that has a special bit **clears** that bit; use the 4-digit form (or symbolic `u+s`/`g+s`/`+t`) to preserve/set it deliberately.
- `find / -perm -4000` (or `-2000`, `-1000`) is the standard audit command for discovering special-bit files on a system.

## Common Mistakes

- Setting `chmod 4755` on a shell script and expecting it to run with elevated privileges — the kernel silently ignores SUID on interpreted scripts.
- Running a routine `chmod 755 file` on something that had `chmod 4755` set, unintentionally stripping the SUID bit because 3-digit mode doesn't preserve special bits.
- Forgetting the sticky bit only restricts **delete/rename**, not read/write of the file's *contents* — assuming it fully isolates users' files from each other in a shared directory when it does not.
- Setting up a shared directory with group permissions but forgetting SGID, so files land with inconsistent groups depending on who created them, breaking team-wide access unpredictably.
- Leaving forgotten SUID-root binaries around from manual builds/experiments, creating an unaudited privilege-escalation surface that a routine `find / -perm -4000` scan would have caught.
