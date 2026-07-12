# Day 7 — Systemd & Cron: Cron & Anacron

**Phase:** 0 – Foundation | **Week:** W1 | **Domain:** Shell | **Flag:**

## Brief

Not everything needs a long-running supervised service — sometimes you just need "run this at 2am every day" or "run this every 15 minutes." That's cron's job, and it predates systemd by decades, but it's still everywhere: backup scripts, log rotation triggers, certificate renewal checks, cache warmers. You need to know cron's syntax cold (it comes up constantly, both in interviews and in real incident postmortems about "why didn't the nightly job run"), and you need to understand its sharp edges — especially the minimal-environment trap that causes cron jobs to fail silently. This file also covers anacron and systemd timers, which exist specifically to fix cron's weaknesses.

## Crontab syntax

```
* * * * * command-to-run
│ │ │ │ │
│ │ │ │ └── day of week (0-7, both 0 and 7 = Sunday)
│ │ │ └──── month (1-12)
│ │ └────── day of month (1-31)
│ └──────── hour (0-23)
└────────── minute (0-59)
```

Field syntax supports:
- `*` — any value.
- `,` — a list: `1,15` = the 1st and 15th.
- `-` — a range: `1-5` = Monday through Friday (as day-of-week).
- `/` — a step: `*/15` = every 15 units (every 15 minutes if in the minute field).

Special strings (shorthand for common schedules, supported by most cron implementations):
```
@reboot     # run once, at startup
@yearly     # 0 0 1 1 *
@annually   # same as @yearly
@monthly    # 0 0 1 * *
@weekly     # 0 0 * * 0
@daily      # 0 0 * * *  (also @midnight)
@hourly     # 0 * * * *
```

`@reboot` is worth calling out specifically: it's cron's substitute for "start on boot," but unlike a systemd service with `Restart=`, it only fires once at boot and has no supervision — if the command dies immediately after, nothing brings it back until the next reboot.

## Editing crontabs

```bash
crontab -e          # edit YOUR user's crontab (opens $EDITOR)
crontab -l           # list your crontab
crontab -r           # remove your entire crontab (no confirmation — careful)
sudo crontab -u deploy -e   # edit another user's crontab (needs root)
```

There's a distinction between **user crontabs** (edited via `crontab -e`, stored per-user under `/var/spool/cron/crontabs/<user>` or similar, and run as that user — no user field in the line) and **system crontabs**:

- `/etc/crontab` — a system-wide file with an extra **user field** between the schedule and the command (`* * * * * root /path/to/script.sh`), because unlike user crontabs it isn't implicitly tied to one owner.
- `/etc/cron.d/*` — drop-in files with the same format as `/etc/crontab` (schedule + user + command). This is the preferred place for **packages** to install their own scheduled jobs without touching a shared file — e.g., a `logrotate` cron entry lives here, not hand-edited into `/etc/crontab`.
- `/etc/cron.daily/`, `/etc/cron.hourly/`, `/etc/cron.weekly/`, `/etc/cron.monthly/` — directories of executable scripts (no schedule syntax needed) run by `run-parts`, triggered by an entry in `/etc/crontab` or by anacron.

## Why cron jobs silently "don't work" — the environment trap

This is the single most common real-world cron bug: **cron runs jobs with a minimal environment**, not your interactive shell's environment. No `.bashrc`, no `.profile`, a bare-bones `PATH` (often just `/usr/bin:/bin`), and a working directory that's typically the user's home directory (not wherever you were standing when you wrote the script). Consequences:

- A script that works perfectly when you run it manually fails under cron because it calls `python3` or `node` and cron's `PATH` doesn't include the directory those binaries live in (e.g., a pyenv/nvm-managed binary in `~/.pyenv/shims` or `~/.nvm/versions/...`).
- Environment variables you rely on (API keys, `$HOME`-dependent config paths) are unset, because cron doesn't source your shell's profile/rc files.
- Relative paths in the script resolve against an unexpected working directory.

The fix pattern practitioners actually use: always use **absolute paths** for both the command and any files it touches, explicitly set `PATH` (and any other needed env vars) at the top of the crontab or inside the script, and **always redirect output** (`>> /var/log/myjob.log 2>&1`) so failures are visible instead of silently dropped (cron mails output to the user's local mailbox by default, which on most modern servers nobody reads or even has configured).

```
# crontab -e example — explicit PATH, absolute paths, logged output
PATH=/usr/local/bin:/usr/bin:/bin
0 2 * * * /usr/bin/python3 /opt/scripts/backup.py >> /var/log/backup.log 2>&1
```

## Anacron — cron for machines that aren't always on

Cron assumes the machine is running continuously — if the box is powered off at 2am when a daily job was scheduled, cron simply never runs it; there's no catch-up. **Anacron** exists specifically for that gap: it's designed for laptops, workstations, or intermittently-running machines, and instead of exact-time scheduling it works in terms of **periods** (in days) since the job last ran. On startup (or on a periodic trigger, itself often invoked by cron or a systemd timer), anacron checks timestamp files in `/var/spool/anacron/` and runs any job whose period has elapsed, even if the machine was off during the "ideal" run time — with a configurable random delay to avoid every job firing at once right after boot. `/etc/anacrontab` defines these jobs (period in days, delay in minutes, job identifier, command) — coarser-grained than cron (days, not minutes), which is fine for its use case of daily/weekly/monthly maintenance jobs, not precise scheduling.

Anacron is a legacy, separate mechanism from cron — it doesn't replace it, it complements it (many distros' `/etc/cron.daily` etc. are actually driven through anacron so the daily/weekly/monthly jobs still run even if the box was off at the scheduled hour).

## Systemd timers as a modern replacement

Systemd timers (`.timer` units, paired with a `.service` unit of the same base name) solve the same "run this on a schedule" problem as cron, but with real supervision:

- **Calendar timers** (`OnCalendar=daily`, `OnCalendar=*-*-* 02:00:00`) work like cron schedules. **Monotonic timers** (`OnBootSec=`, `OnUnitActiveSec=`) schedule relative to an event (time since boot, time since the unit last activated) — something cron simply cannot express, since cron only understands wall-clock schedules.
- **`Persistent=true`** in the `[Timer]` section is systemd's built-in answer to anacron's problem: if set, systemd records the last trigger time and, if the machine was off when the timer should have fired, runs it once at the next boot to catch up — no separate anacron mechanism needed.
- Because the triggered unit is a normal `.service`, you get full journal integration (`journalctl -u myjob.service`), dependency ordering (`After=`/`Requires=`), and resource controls (cgroups, `MemoryMax=`, etc.) — none of which cron gives you.
- List and inspect timers with `systemctl list-timers` (shows next/last run times for every timer on the box) — there's no cron equivalent that shows "when will this actually run next" so directly.

This is why many teams migrate recurring jobs off cron and onto systemd timers over time: same scheduling power (and more, via monotonic timers), plus logging, dependency management, and missed-run catch-up in one consistent tool, instead of stitching cron + anacron + shell redirection + a separate log file together.

## Points to Remember

- Crontab fields are minute, hour, day-of-month, month, day-of-week, in that order; `*/N` is a step, `,` is a list, `-` is a range.
- `@reboot` runs once at startup with no supervision — it is not a substitute for `Restart=` on a systemd service.
- User crontabs (`crontab -e`) have no user field; system crontabs (`/etc/crontab`, `/etc/cron.d/*`) do, because they aren't implicitly owned by one user.
- Cron jobs run with a minimal `PATH` and no shell profile sourced — always use absolute paths and set `PATH` explicitly.
- Anacron is period-based (days) and designed to catch up missed runs on machines that aren't always powered on; cron has no catch-up mechanism of its own.
- `Persistent=true` on a systemd timer gives you anacron-like catch-up behavior natively, without needing anacron as a separate tool.

## Common Mistakes

- Writing a script that works fine manually, adding it to cron, and having it fail silently because `python3`/`node`/a custom binary isn't on cron's minimal `PATH` — and not noticing because output wasn't redirected anywhere.
- Assuming `@reboot` gives crash-resilience like a systemd service — it fires once at boot only; if the process dies five minutes later, nothing restarts it.
- Editing `/etc/crontab` directly for a package-specific job instead of dropping a file in `/etc/cron.d/` — makes upgrades/packaging conflicts more likely and is harder to manage per-package.
- Forgetting `crontab -l` is per-user and per-host — checking the wrong user's crontab (or the wrong server) and concluding "the cron job isn't there" when it's just under a different user.
- Relying on cron's default mail-on-output behavior to catch failures, on a server where local mail was never configured — errors vanish into an unread mailbox instead of a log file.
- Assuming anacron replaces cron entirely — it's coarser-grained (day-level) and meant for maintenance-style jobs, not minute-level scheduling; conflating the two leads to writing an anacrontab entry expecting minute-precision timing it can't provide.
