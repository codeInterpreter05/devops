# Day 7 — Systemd & Cron: journalctl & Logging

**Phase:** 0 – Foundation | **Week:** W1 | **Domain:** Shell | **Flag:**

## Brief

Before systemd, logs were whatever a daemon chose to write, wherever it chose to write it — usually flat text files under `/var/log`, one convention per program, parsed with `grep`/`tail`/`awk`. Systemd replaces (or supplements) that with `journald`, a logging daemon that captures stdout/stderr and kernel messages from every unit into a single structured, indexed store, queried with `journalctl`. When your Python service in the previous file crashes, `journalctl` is where you go to find out why — it's the log tool you'll use every single day operating Linux services, and interviewers expect you to know its filtering flags cold.

## Why a structured journal instead of flat text files

`journald` stores log entries as structured records (binary format, not plain text) with rich metadata attached automatically: which unit/PID/UID generated the message, a monotonic and wall-clock timestamp, boot ID, priority level, and more. This buys you things `grep`ing a flat file can't give you cheaply:

- **Filter by unit** without needing the app to tag its own lines (`journalctl -u nginx`).
- **Filter by exact boot** (`journalctl -b`, `journalctl -b -1` for the previous boot) — a flat file that rotates on reboot loses this distinction, or you have to grep timestamps.
- **Filter by priority** (`-p err`) without every app agreeing on a log-level string format.
- **Binary indexing** makes range queries (by time) fast even on large logs, versus scanning a multi-GB text file linearly.

The tradeoff: journal files (`/var/run/log/journal/` or `/var/log/journal/`, see below) are binary and not `grep`-friendly directly — you go through `journalctl`, or export with `journalctl -o json`/`--output=export` if you need to pipe structured data elsewhere (e.g., to a log shipper).

## Persistent vs. volatile journal

By default on many distros the journal is **volatile** — stored in `/run/log/journal/` (tmpfs, memory-backed), which means **logs are lost on reboot**. Whether this is the case depends on whether `/var/log/journal/` exists:

- If `/var/log/journal/` exists, journald stores there and logs **persist across reboots** (and survive until rotated out by size/time limits in `journald.conf`).
- If it doesn't exist, journald falls back to `/run/log/journal/` (RAM-backed, wiped on reboot).

To make logging persistent:
```bash
sudo mkdir -p /var/log/journal
sudo systemd-tmpfiles --create --prefix /var/log/journal
sudo systemctl restart systemd-journald
```
This is a real gotcha in fresh cloud images and minimal containers — you debug a crash, reboot to try a fix, and find `journalctl -b -1` returns nothing because the previous boot's logs were never persisted anywhere.

## `journalctl` — the query tool

```bash
journalctl -u myapp.service         # everything logged by this unit, oldest first
journalctl -u myapp -f              # follow / tail -f equivalent, live stream
journalctl -u myapp -n 50           # last 50 lines
journalctl -u myapp --since "10 min ago"
journalctl -u myapp --since "2026-07-12 09:00" --until "2026-07-12 10:00"
journalctl -u myapp -p err          # priority filter: emerg,alert,crit,err,warning,notice,info,debug
journalctl -b                        # logs from current boot only
journalctl -b -1                     # logs from the previous boot
journalctl -k                        # kernel messages only (dmesg equivalent, via the journal)
journalctl --disk-usage              # how much space the journal is using
journalctl --vacuum-time=7d          # delete entries older than 7 days
journalctl --vacuum-size=500M        # shrink journal to a max size
journalctl -o json-pretty -u myapp   # structured output for scripting/shipping
```

`-p err` deserves a callout: priority filters in the journal are **"this level and more severe"** by default, not an exact match — `journalctl -p err` shows `err`, `crit`, `alert`, and `emerg` entries, not only `err`. If you want an exact single level, you need the range syntax `journalctl -p err..err`.

`--since`/`--until` accept both relative ("2 hours ago", "yesterday") and absolute (`"2026-07-12 09:00:00"`) formats — combine them to bound a search window when investigating an incident, which is far faster than eyeballing a multi-thousand-line unfiltered dump.

## Connecting logs back to `Restart=`

When a service configured with `Restart=on-failure` crashes and comes back up, `journalctl -u myapp` will show the crash's stderr output, then a systemd-generated line noting the process exited and the unit is scheduled to restart, then the new process's startup output — all interleaved in one place with a boot-scoped, time-ordered view. This is why `journalctl -u <service> -f` is the standard "watch it restart in real time" command during the lab exercise: you don't need to separately tail an app log file and separately check `systemctl status` — the journal already correlates both.

## Points to Remember

- `journald` stores structured, indexed log records (unit, PID, UID, priority, boot ID, timestamps) — not flat text — which is what makes `-u`, `-b`, and `-p` filtering possible without app cooperation.
- Journal persistence depends on whether `/var/log/journal/` exists; without it, logs live in RAM (`/run/log/journal/`) and vanish on reboot.
- `journalctl -p <level>` matches that level **and everything more severe**, not an exact match, unless you use `level..level` range syntax.
- `journalctl -u <unit> -f` is the live-tail equivalent for a specific service — the single most-used command when debugging a service in real time.
- `--vacuum-time=`/`--vacuum-size=` control disk usage; an unbounded persistent journal can fill a disk on a chatty service.

## Common Mistakes

- Rebooting a box to test a fix, then being confused that `journalctl -b -1` shows nothing — persistent journaling was never enabled (`/var/log/journal/` doesn't exist).
- Assuming `journalctl -p err` shows *only* error-level lines, then missing that it's also including `crit`/`alert`/`emerg` (usually harmless, but can mislead severity counts).
- Running `journalctl` with no filters on a long-lived box and getting flooded with output from every unit since the journal began, instead of scoping with `-u`, `-b`, or `--since`.
- Not knowing `--vacuum-size`/`--vacuum-time` exist, and instead manually deleting journal files under `/var/log/journal/` (risky — can corrupt an in-use journal; always go through `journalctl` or `systemctl restart systemd-journald` first if you must touch the files directly).
- Forgetting `-f` needs `sudo` (or journal group membership) to follow logs for services running as another user, and assuming the command "isn't working" when it's actually a permissions issue silently returning partial output.
