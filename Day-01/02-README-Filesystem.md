# Day 1 — Linux Fundamentals I: Filesystem Hierarchy

**Phase:** 0 – Foundation | **Week:** W1 | **Domain:** Linux | **Flag:** ⚡ Interview-critical

## Brief

This is the single highest-yield interview topic in this phase — "explain the Linux filesystem hierarchy and where configs/logs live" is asked constantly, because it's a fast way to tell if a candidate has *actually operated* Linux systems or only used them passively. This note covers the Filesystem Hierarchy Standard (FHS) and the absolute-vs-relative path distinction that trips up scripts constantly.

## The single-tree model

Unlike Windows (`C:\`, `D:\`), Linux has **one tree, rooted at `/`**. Every disk, partition, USB stick, network share, or virtual filesystem gets **mounted** onto a directory somewhere in that tree (see `mount`, `/etc/fstab`, Day 18 for depth). There's no such thing as "which drive is this file on" from the user's perspective — just a path.

## FHS — what lives where

| Path | Purpose | Notes |
|---|---|---|
| `/` | Root of everything | Everything hangs off this |
| `/bin`, `/sbin` | Essential user/admin binaries | On modern distros usually symlinked into `/usr/bin`, `/usr/sbin` |
| `/usr` | Secondary hierarchy — most installed software lives here | `/usr/bin`, `/usr/lib`, `/usr/share` |
| `/usr/local` | Software you compiled/installed manually, outside the package manager | Package managers won't touch this — safe zone for manual installs |
| `/etc` | System-wide **configuration files** | No binaries here, just configs. `/etc/passwd`, `/etc/hosts`, `/etc/nginx/nginx.conf` |
| `/var` | **Variable data** — things that grow/change at runtime | Logs, databases, mail spools, package caches |
| `/var/log` | **Logs** | `/var/log/syslog`, `/var/log/auth.log`, app logs |
| `/var/lib` | Persistent application state | Docker's data lives at `/var/lib/docker` |
| `/home` | Per-user home directories | `/home/ubuntu`, `/home/deploy` |
| `/root` | The root user's home directory | Not inside `/home` — special-cased |
| `/tmp` | Temporary files | Often cleared on reboot or by a timer — never store anything you need to keep |
| `/opt` | Optional/third-party packaged software | Common for vendor software bundles, e.g. `/opt/datadog-agent` |
| `/proc` | **Virtual** filesystem exposing kernel/process info | Not real files on disk — read `/proc/cpuinfo`, `/proc/meminfo`, `/proc/<pid>/status` live |
| `/sys` | Virtual filesystem exposing kernel/device/driver state | Used for things like tuning kernel parameters at runtime |
| `/dev` | Device files | `/dev/sda`, `/dev/null`, `/dev/random` |
| `/mnt`, `/media` | Manual and removable-media mount points | `/mnt` for admin-mounted filesystems, `/media` for auto-mounted USB/CD |
| `/boot` | Kernel, initramfs, bootloader files | What GRUB loads at boot |

**The interview-critical takeaway:** configs live in `/etc`, logs live in `/var/log`, persistent app state lives in `/var/lib`, and `/proc` + `/sys` are *not real files* — they're the kernel exposing live state through a filesystem interface (which is why `cat /proc/cpuinfo` returns instantly and a size of "0 bytes" doesn't mean it's empty).

## Absolute vs. relative paths

- **Absolute path**: starts with `/`, unambiguous regardless of where you are. `/etc/nginx/nginx.conf` always means the same file.
- **Relative path**: interpreted relative to your **current working directory** (`pwd`). `nginx.conf` or `../nginx/nginx.conf` means something different depending on where you're standing.
- Special relative tokens: `.` (current dir), `..` (parent dir), `~` (your home directory — expanded by the *shell*, not the kernel, so it won't work inside quotes passed to programs that don't do their own expansion, or in most non-interactive contexts like some cron entries).

**Why this matters operationally:** scripts and cron jobs / systemd services often start with an unexpected working directory (frequently `/` or `/root`). A script that does `rm -rf ./output` assuming it's run from `/home/deploy/app` will do something very different if invoked from `/`. Always `cd` explicitly to a known directory at the top of a script, or use absolute paths throughout.

## Points to Remember

- One tree, one root (`/`) — no drive letters. Everything is mounted into this tree.
- `/etc` = configs, `/var/log` = logs, `/var/lib` = persistent app state, `/tmp` = ephemeral (may vanish on reboot), `/opt` = third-party app bundles.
- `/proc` and `/sys` are virtual — they don't consume disk space and reflect live kernel/process state, not stored files.
- Absolute paths are safe everywhere; relative paths depend on `pwd` and are the #1 cause of scripts that "work on my machine" but fail in cron/systemd/CI.
- `~` expansion is a **shell** feature — don't assume it works in every context (e.g., some non-interactive shells, or when a path is passed as a literal string to an API).

## Common Mistakes

- Writing deploy/backup scripts with relative paths and no explicit `cd`, then having them silently do the wrong thing when triggered by cron (which has a minimal environment and often starts in `$HOME` or `/`).
- Confusing `/var/lib` (persistent state, should be backed up) with `/var/cache` or `/tmp` (disposable) — deleting the wrong one during "cleanup" can destroy application data (e.g., wiping `/var/lib/docker` or a database's data directory).
- Assuming `/proc/<pid>` will still exist by the time you act on it — processes can exit between when you list them and when you read their `/proc` entry (classic TOCTOU race in shell scripts that parse `/proc`).
- Not knowing `/usr/local` exists as the "manual install" zone — installing custom-compiled software into `/usr/bin` directly, which package managers may later overwrite or conflict with.
