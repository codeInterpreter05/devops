# Day 2 — Resources: Linux Fundamentals II

## Primary (assigned)

- **Linux Journey — Users section** (linuxjourney.com) — free, interactive, the assigned starting point for this day. Covers users, groups, and permissions in a beginner-friendly, hands-on way that pairs well with the labs above.

## Deepen your understanding

- **`man chmod`, `man chown`, `man sudoers`, `man passwd`** — the authoritative reference for every flag and sudoers directive covered today; run these directly on any Linux box, no internet required.
- **DigitalOcean — "Linux Permissions Basics and How to Use Umask on Servers"** — a widely used, clearly written walkthrough of the rwx model and `umask` with practical server-administration framing.
- **sudo project documentation** (sudo.ws/docs) — the canonical reference for sudoers syntax, aliases, and the full range of directives beyond what's covered here (command aliases, `Defaults` lines, digest specs).
- **Julia Evans' zines** (jvns.ca) — short, illustrated deep-dives; several cover permissions and processes in a way that builds real intuition rather than rote memorization.

## Reference / lookup

- **`man -k passwd`** (equivalent to `apropos passwd`) — quickly surfaces every related man page (`passwd`, `passwd(5)`, `chpasswd`, `vipw`) when you're not sure which section you need.
- **explainshell.com** — paste any `chmod`/`chown`/`useradd` invocation and get every flag broken down inline.

## Practice

- **OverTheWire: Bandit** (overthewire.org/wargames/bandit) — several early levels directly exercise file permissions, SUID binaries, and privilege boundaries via SSH-based challenges — excellent hands-on reinforcement of everything in today's lab.
- **A disposable Docker container or VM** — the fastest, lowest-risk way to practice `useradd`/`usermod`/`visudo`/SUID experiments repeatedly without any risk to a real system; spin one up, break things, destroy it, repeat.
