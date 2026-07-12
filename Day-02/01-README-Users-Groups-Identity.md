# Day 2 — Linux Fundamentals II: Users, Groups & Identity

**Phase:** 0 – Foundation | **Week:** W1 | **Domain:** Linux | **Flag:** ⚡ Interview-critical

## Brief

Every process on a Linux box runs *as* somebody — a user, and by extension one or more groups. That identity is the entire basis for access control: what files you can read, what services you can start, whether you can bind to port 80. In DevOps this shows up everywhere: service accounts running your app as a non-root UID in a container, CI runners needing scoped permissions, shared build servers where multiple engineers' cron jobs must not step on each other's files. Interviewers probe this because "explain what happens when a user logs in" or "what's the difference between a user's primary and secondary group" separates people who've only used `sudo` reflexively from people who understand *why* it works.

This day is split into four focused files:

1. **This file** — the user/group identity model: `/etc/passwd`, `/etc/shadow`, `/etc/group`, and the `id`/`whoami` family of commands.
2. **[02-README-Permissions-Ownership.md](02-README-Permissions-Ownership.md)** — the rwx permission model, `chmod`/`chown` in depth, and how ownership is assigned.
3. **[03-README-Sudo-Sudoers.md](03-README-Sudo-Sudoers.md)** — `sudo` vs `su`, `/etc/sudoers`, and privilege escalation mechanics.
4. **[04-README-Special-Bits.md](04-README-Special-Bits.md)** — SUID, SGID, and the sticky bit.

## The user/group model

Internally, the kernel doesn't know usernames — it knows **numeric IDs**. A **UID** (user ID) identifies a user; a **GID** (group ID) identifies a group. Usernames like `alice` are a human-friendly lookup layer resolved via `/etc/passwd` (or LDAP/AD/SSSD in enterprise setups, configured through `/etc/nsswitch.conf`). Every process has, at minimum, a **real UID/GID** (who invoked it) and an **effective UID/GID** (whose permissions are actually checked) — these are usually the same, except during privilege escalation (covered in the SUID note).

- **UID 0 is always root**, regardless of username. Rename `root` to something else and UID 0 still has full privileges — the number is what matters, not the name.
- **System/service UIDs** (historically 1–999, sometimes 1–99 for "system" vs 100–999 for services depending on distro) are reserved for daemons and services (`www-data`, `postgres`, `sshd`) — they typically have no login shell (`/usr/sbin/nologin` or `/bin/false`) and no real password, because nobody should interactively log in as them.
- **Regular human users** start at UID 1000 on Debian/Ubuntu (500 or 1000 depending on RHEL version) — this is configured in `/etc/login.defs` (`UID_MIN`/`UID_MAX`).

## `/etc/passwd` — the identity database

```
alice:x:1001:1001:Alice Smith,,,:/home/alice:/bin/bash
```

Seven colon-separated fields: **username : password-placeholder : UID : GID (primary group) : GECOS (comment/full name) : home directory : login shell**. The `x` in the password field means "the real hash lives in `/etc/shadow`, not here" — `/etc/passwd` must remain **world-readable** (many tools resolve UID→name by reading it), so storing password hashes there directly, as ancient Unix once did, exposed every hash to any local user for offline cracking.

## `/etc/shadow` — actual credentials

Root-only readable (`-rw-------`, owner `root`), fields include: `username:hash:last-changed:min-days:max-days:warn-days:inactive-days:expire-date`. The hash field also encodes the algorithm — `$6$...` is SHA-512, `$y$...` is yescrypt (modern default on many distros), `!` or `*` means the account has no valid password (login via password disabled, common for service accounts). This split — public metadata in `passwd`, secrets in `shadow` — is itself a small but real lesson in least privilege: don't put secrets somewhere that has to be world-readable for unrelated reasons.

## `/etc/group` — group membership

```
devteam:x:1010:alice,bob
```

**Group name : password placeholder (rarely used, see `/etc/gshadow`) : GID : comma-separated list of secondary/supplementary members.** Crucially, a user's **primary group** (the GID field in `/etc/passwd`) does **not** need to appear in `/etc/group`'s member list — primary group membership is implicit via `/etc/passwd`, while `/etc/group` only lists **supplementary (secondary)** memberships. This trips people up when grepping `/etc/group` for a user and not finding their primary group listed there at all.

- **Primary group**: the group assigned to files a user creates by default (unless SGID or `newgrp` changes this — see the special-bits note).
- **Secondary groups**: additional groups a user belongs to that grant extra access (e.g., `docker`, `sudo`, `devteam`) without changing what group new files get by default.
- Modern distros default to **UPG (User Private Group)**: `useradd` creates a group with the same name and GID as the new user (e.g., user `alice`, primary group `alice`) rather than dumping everyone into a shared `users` group. This makes default file ownership more predictable per-user while `umask` still controls what group members can do with those files.

## Identity commands

```bash
whoami           # just the effective username
id               # full identity: uid, gid, and all groups
id -u            # numeric UID only
id -g            # numeric primary GID only
id -nG           # names of ALL groups (primary + secondary) the user belongs to
id alice         # identity info for a specific user, not just yourself
groups           # shorthand for group membership (uses NSS, same source as id)
who am i         # who is logged into THIS terminal session (from utmp) — differs from whoami after su
logname          # the ORIGINAL login name, even after su/sudo change effective identity
getent passwd alice   # look up a user via NSS (works whether backed by /etc/passwd, LDAP, or SSSD)
```

`whoami` and `who am i` look similar but answer different questions: `whoami` reports your **current effective user** (changes after `su`), while `who am i` and `logname` report the **originally authenticated login session** — useful in audit trails to answer "who actually logged in, regardless of how many times they `su`'d afterward."

## Creating and managing users/groups

```bash
useradd -m -s /bin/bash alice     # -m creates home dir, -s sets login shell
passwd alice                      # interactively set/change password
usermod -aG devteam alice         # ADD alice to devteam as a secondary group (-a = append, never omit it)
usermod -s /usr/sbin/nologin svc  # change login shell (e.g., disable interactive login for a service account)
groupadd devteam                  # create a new group
userdel -r alice                  # delete user AND home directory/mail spool
groupdel devteam                  # delete a group (fails if it's still someone's primary group)
```

The `-a` flag on `usermod -aG` is the single most consequential detail here: `usermod -G devteam alice` **without** `-a` *replaces* alice's entire supplementary group list with just `devteam`, silently removing her from `docker`, `sudo`, or anything else she was in. This is a well-known footgun.

## Points to Remember

- The kernel checks **UIDs/GIDs**, not names — `root` is "whichever account has UID 0," full stop.
- `/etc/passwd` is world-readable metadata; `/etc/shadow` holds the actual password hashes and is root-only — this split exists so ordinary lookups (UID→name) don't require exposing secrets.
- A user's **primary group** lives in `/etc/passwd` (GID field) and is *not* listed in `/etc/group`'s member list; `/etc/group` only shows **secondary** memberships.
- `id -nG` shows every group a user belongs to (primary + secondary) — the most reliable one-liner for "what can this account access via group membership."
- `whoami`/`id` reflect your *current effective* identity; `logname`/`who am i` reflect who *originally* authenticated — they diverge after `su`.

## Common Mistakes

- Running `usermod -G group user` (no `-a`) intending to *add* a group, and instead wiping out all of that user's other group memberships — a classic way to silently break someone's `sudo` or `docker` access.
- Confusing "user has no entry in `/etc/group`" with "user isn't in that group" — forgetting to also check whether the group is their *primary* group via `/etc/passwd`.
- Deleting a user with plain `userdel` (no `-r`) and leaving an orphaned home directory and mail spool behind, or the reverse — using `-r` on an account whose home directory is shared/mounted storage other things depend on.
- Assuming `/etc/passwd` is where passwords live because of the name — it hasn't stored real password hashes by default since shadow passwords became standard decades ago.
- Creating service accounts with a normal login shell and a real password instead of `/usr/sbin/nologin` and a locked (`!`/`*`) shadow entry, needlessly widening the attack surface.
