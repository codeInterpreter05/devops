# Day 1 — Linux Fundamentals I: Overview

**Phase:** 0 – Foundation | **Week:** W1 | **Domain:** Linux | **Flag:** ⚡ Interview-critical

## Brief

Everything in DevOps eventually lands on a Linux box — a CI runner, a Kubernetes node, an EC2 instance, a container's root filesystem. If you don't have a solid mental model of *how Linux organizes itself*, every layer built on top of it (containers, orchestration, cloud) feels like memorized trivia instead of understanding. Today builds that base: what Linux actually is, how to get help without Googling everything, and the map you'll need before you can navigate it (covered in depth in the next two notes).

This day is split into three focused files:

1. **This file** — what Linux is, distro landscape, and how to get help (`man`, `tldr`).
2. **[02-README-Filesystem.md](02-README-Filesystem.md)** — the Filesystem Hierarchy Standard (FHS): what lives where and why, plus absolute vs. relative paths.
3. **[03-README-Essential-Commands.md](03-README-Essential-Commands.md)** — `ls`, `cd`, `cp`, `mv`, `rm`, `find` in depth.

## What Linux actually is

Linux is a **kernel** — the piece of software that talks to hardware, manages processes, memory, and files. What people call "Linux" day-to-day (Ubuntu, Debian, RHEL, Alpine, Amazon Linux) is really **GNU/Linux**: the Linux kernel plus a userland of GNU tools (bash, coreutils, etc.) plus a package manager and init system, bundled by a distribution.

Why this matters for DevOps:
- **Alpine** (musl libc, ~5MB base) is popular for containers because it's tiny — but its C library differences occasionally break binaries compiled against glibc. This is *the* classic "works on Ubuntu, breaks in my Alpine-based container image" bug.
- **Ubuntu/Debian** use `apt`; **RHEL/CentOS/Amazon Linux/Fedora** use `yum`/`dnf`. Same Linux kernel, different package managers and file layout conventions in places — know which family you're on before you paste install commands from a tutorial.
- **Amazon Linux 2/2023** is the default for EC2/many managed AWS services — worth being comfortable in it specifically since that's likely what you'll SSH into professionally.

## Getting help without Googling everything

Two tools you should reach for before a search engine:

- **`man <command>`** — the manual page. Structured into numbered sections (1 = user commands, 5 = file formats, 8 = admin commands). `man 5 crontab` gets you the crontab *file format* docs, while `man 1 crontab` (or just `man crontab`) gets the command. Search inside a man page with `/searchterm` then `n` for next match.
- **`tldr <command>`** — community-maintained, example-first cheat sheets. `tldr tar` gives you five practical one-liners instead of tar's notoriously dense man page. Install via `npm install -g tldr`, `pip install tldr`, or your package manager.

Workflow that actually works under pressure: **`tldr` first to jog your memory with a concrete example, `man` when you need a flag `tldr` didn't cover or need to be 100% sure of exact behavior.**

## Points to Remember

- "Linux" = kernel; the distro is kernel + GNU userland + package manager + init system. Different distros can behave differently even though "it's all Linux."
- Know your distro family (Debian/Ubuntu vs RHEL/Fedora vs Alpine) before running install commands — package names and manager syntax differ (`apt install` vs `dnf install` vs `apk add`).
- `man` sections matter: the same word can mean a command (section 1) or a config file format (section 5). `man -k <keyword>` searches all man page descriptions (equivalent to `apropos`).
- `tldr` is not a replacement for `man` — it's a faster on-ramp. For anything safety-critical (permissions, `dd`, `rm`), confirm behavior in `man` before running.

## Common Mistakes

- Blindly copy-pasting shell commands from a tutorial written for a different distro family (e.g., running `apt` commands on Amazon Linux, which uses `yum`/`dnf`).
- Assuming glibc-only binaries will "just work" in an Alpine-based Docker image — they often won't, because Alpine uses musl libc. If you hit `not found` errors for a binary that clearly exists, this is the first thing to suspect.
- Not knowing `man`'s section numbers exist, then being confused when `man crontab` doesn't show the file-format details you needed (that's `man 5 crontab`).
- Treating `tldr` output as exhaustive — it deliberately omits edge-case flags. Don't use it as your only source when the command is destructive (`rm`, `dd`, `mkfs`).
