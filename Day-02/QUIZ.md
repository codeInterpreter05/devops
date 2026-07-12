# Day 2 — Quiz: Linux Fundamentals II

Try to answer without looking at your notes. Answers are at the bottom.

1. What's the difference between a user's **primary** group and their **secondary (supplementary)** groups, and where is each one actually stored?
2. Why does real password hash data live in `/etc/shadow` instead of `/etc/passwd`, given that `/etc/passwd` also stores a "password" field?
3. What does the execute (`x`) bit mean on a **directory**, as opposed to on a file? Give a concrete scenario where this distinction matters.
4. Explain why you can sometimes delete a file you don't own and have no write permission on.
5. What is the practical difference between `su` and `sudo` in terms of whose password is required and what gets logged?
6. Why does `su -` behave differently from plain `su`?
7. What happens if you run `usermod -G devteam alice` (no `-a` flag) when alice is already in the `docker` group? Why?
8. What does the SGID bit do when set on a **directory**, and what real-world problem does it solve?
9. Why does `chmod 4755 myscript.sh` fail to give a shell script elevated privileges, even though the SUID bit is set correctly?
10. **Interview question:** What is the difference between SUID and sticky bit? When would you use each?
11. Why must `/etc/sudoers` always be edited with `visudo` rather than a normal text editor?
12. If a numeric `chmod` mode has 4 digits (e.g. `4750`) instead of 3 (e.g. `750`), what does the leading digit control, and what happens if you later run a plain 3-digit `chmod` on that same file?

---

## Answers

1. The **primary** group is recorded in `/etc/passwd` (the GID field) and is the group assigned by default to files the user creates. **Secondary/supplementary** groups are listed in `/etc/group`'s member list and grant additional access without changing what group new files get by default. A user's primary group typically does **not** appear in `/etc/group`'s member list at all — it's implicit via `/etc/passwd`.
2. `/etc/passwd` must remain world-readable because many tools resolve UID→username by reading it. Storing real password hashes in a world-readable file would let any local user copy every hash for offline cracking. `/etc/shadow` is root-only readable, so hashes stay protected while lookups that don't need secrets still work against `/etc/passwd`.
3. On a directory, execute means **traverse** — the ability to `cd` into it or access anything inside via a full path, even if you know the exact filename. Read on a directory means you can **list** its contents. A directory with `r--` (read, no execute) lets you see filenames via `ls` but not access anything inside by path; a directory with `--x` (execute, no read) lets you access a file directly if you already know its exact name, but you can't list what's there.
4. Deleting a file is a **directory** operation, not a file operation — it requires write permission on the **containing directory**, not on the file itself. If the directory is writable to you (and doesn't have the sticky bit set), you can delete any file inside regardless of that file's own permissions or owner.
5. `su` switches your entire session to another user and requires **that target user's own password** (default target: root); it's logged as a single session-switch event. `sudo` runs one command as another user (default: root) using **your own password**, and every invocation is logged individually (typically to `/var/log/auth.log` or `/var/log/secure`) — giving a per-command audit trail that `su` doesn't provide.
6. `su -` (or `su -l`) starts a full **login shell**, resetting environment variables (`PATH`, `HOME`, etc.) to the target user's own login environment. Plain `su` keeps your **current** shell's environment, which can cause subtle bugs — e.g., your own `PATH` (potentially pointing at directories you control) staying in effect even while "acting as" another user.
7. It **replaces** alice's entire supplementary group list with just `devteam`, silently removing her from `docker` and anything else she was in — because `-G` (without `-a`) sets the group list exactly rather than appending. The correct command is `usermod -aG devteam alice`.
8. SGID on a directory makes every file/subdirectory created inside **inherit the directory's group**, instead of the creating user's own primary group (and typically propagates the SGID bit to new subdirectories too). This solves the shared-team-directory problem: without it, files created by different team members would land with different, inconsistent groups, breaking uniform group-based access.
9. Modern Linux kernels deliberately **ignore the SUID bit on scripts** (anything invoked via a shebang/interpreter) due to a historical race-condition vulnerability class between the kernel opening the script and the interpreter re-reading it. SUID only reliably elevates privilege on **compiled binaries**, not interpreted scripts.
10. **Interview question answer:** SUID changes the **effective UID of a running process** to the file's owner — it's about controlled privilege elevation during execution of a trusted binary (e.g., `passwd`, letting any user update their own entry in root-only `/etc/shadow`). The sticky bit is unrelated to execution or privilege — applied to a shared, world-writable **directory**, it restricts deletion/renaming so only a file's owner (or root) can remove it, even though the directory itself is writable by everyone (e.g., `/tmp`, mode `1777`). Use SUID when an unprivileged user needs to perform one specific privileged action through a trusted program; use the sticky bit when a directory must be writable by many users without letting them delete each other's files.
11. `visudo` locks the sudoers file against simultaneous edits and, critically, **validates syntax before saving** — a malformed sudoers file edited directly with a plain editor can break `sudo` for every user on the box, potentially requiring single-user-mode/console recovery to fix.
12. The leading (4th) digit controls the **special bits** — SUID (4), SGID (2), sticky (1), or a sum of them. A plain 3-digit `chmod` (e.g. `chmod 750`) sets the standard rwx bits but implicitly **clears** any special bit that was previously set, since it doesn't specify a 4th digit at all — a common way SUID/SGID/sticky bits silently disappear after a routine permissions "fix."
