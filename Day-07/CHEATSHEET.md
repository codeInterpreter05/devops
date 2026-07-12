# Day 7 — Cheatsheet: Systemd & Cron

## Unit file locations (precedence, highest wins)

```
/etc/systemd/system/        admin-created / overrides — highest precedence
/run/systemd/system/        runtime-generated, volatile
/usr/lib/systemd/system/    package-installed defaults — lowest precedence
```

## `.service` unit skeleton

```ini
[Unit]
Description=short description shown in status/logs
After=network-online.target        # ordering only
Wants=network-online.target        # soft dependency
Requires=some-other.service        # hard dependency (this fails if that fails)

[Service]
Type=simple                        # simple | forking | oneshot | notify
ExecStart=/usr/bin/python3 /opt/app/main.py
ExecStartPre=/bin/echo starting
ExecStop=/bin/kill -TERM $MAINPID
Restart=on-failure                 # no|on-success|on-failure|on-abnormal|on-watchdog|on-abort|always
RestartSec=5
StartLimitIntervalSec=60
StartLimitBurst=5
TimeoutStartSec=30
User=appuser
WorkingDirectory=/opt/app
Environment=KEY=value

[Install]
WantedBy=multi-user.target
```

## `Restart=` policy quick reference

```
no             never restart automatically (default)
on-success     restart only on clean exit (code 0 / clean signal)
on-failure     restart on non-zero exit, signal kill, or timeout  <- standard "restart on crash"
on-abnormal     restart on signal/timeout/watchdog, not on plain non-zero exit
on-watchdog    restart only if WatchdogSec= expires (missed keepalive)
on-abort        restart only on uncaught fatal signal (e.g. segfault)
always          restart no matter how/why it exited, including clean stop via kill (not `systemctl stop`)
```

## `.timer` unit skeleton (cron replacement)

```ini
# myjob.timer
[Timer]
OnCalendar=*-*-* 02:00:00      # calendar/wall-clock schedule, like cron
OnBootSec=5min                  # monotonic: 5 min after boot
OnUnitActiveSec=1h              # monotonic: 1h after this unit last activated
Persistent=true                 # catch up a missed run after downtime (anacron-like)

[Install]
WantedBy=timers.target
```
Pair with a same-named `.service` (`myjob.service`, typically `Type=oneshot`). Enable the `.timer`, not the `.service`.

## `systemctl`

```bash
systemctl start <unit>              # start now
systemctl stop <unit>               # stop now
systemctl restart <unit>            # stop + start
systemctl reload <unit>             # reload config without restart (if supported)
systemctl enable <unit>             # persist across boot (does NOT start now)
systemctl enable --now <unit>       # enable + start in one shot
systemctl disable <unit>            # remove boot persistence (does NOT stop now)
systemctl status <unit>             # state + recent logs + main PID
systemctl is-active <unit>          # active|inactive (script-friendly)
systemctl is-enabled <unit>         # enabled|disabled (script-friendly)
systemctl cat <unit>                # print the unit file(s) systemd actually loaded
systemctl show -p MainPID <unit>    # query a specific unit property
systemctl daemon-reload             # REQUIRED after editing any unit file
systemctl list-units --type=service # list active service units
systemctl list-timers               # list all timers + next/last run time
systemctl list-unit-files           # list all known units + enabled state
systemctl reset-failed <unit>       # clear a "failed" state / restart-limit trip
```

## `journalctl`

```bash
journalctl -u <unit>                 # all logs for a unit
journalctl -u <unit> -f              # follow / live tail
journalctl -u <unit> -n 50           # last 50 lines
journalctl -u <unit> --since "10 min ago"
journalctl -u <unit> --since "2026-07-12 09:00" --until "2026-07-12 10:00"
journalctl -u <unit> -p err           # priority filter: this level + more severe
journalctl -u <unit> -p err..err      # exact level only
journalctl -b                         # current boot only
journalctl -b -1                      # previous boot
journalctl -k                         # kernel messages only
journalctl -o json-pretty -u <unit>   # structured output
journalctl --disk-usage               # journal size on disk
journalctl --vacuum-time=7d           # drop entries older than 7 days
journalctl --vacuum-size=500M         # cap journal size
```
Priority levels (most to least severe): `emerg alert crit err warning notice info debug`

Make journal persistent (survive reboot):
```bash
sudo mkdir -p /var/log/journal
sudo systemd-tmpfiles --create --prefix /var/log/journal
sudo systemctl restart systemd-journald
```

## Crontab syntax

```
*  *  *  *  *  command
│  │  │  │  │
│  │  │  │  └── day of week (0-7, 0 and 7 both = Sunday)
│  │  │  └───── month (1-12)
│  │  └──────── day of month (1-31)
│  └─────────── hour (0-23)
└────────────── minute (0-59)

,   list        1,15 * * * *      -> 1st and 15th minute
-   range       * 9-17 * * 1-5    -> hourly, 9am-5pm, Mon-Fri
/   step        */15 * * * *      -> every 15 minutes
```

Special strings:
```
@reboot     run once at startup (no supervision/restart)
@yearly     0 0 1 1 *
@monthly    0 0 1 * *
@weekly     0 0 * * 0
@daily      0 0 * * *   (alias: @midnight)
@hourly     0 * * * *
```

## `crontab` command

```bash
crontab -e                    # edit your own crontab
crontab -l                    # list your own crontab
crontab -r                    # remove your ENTIRE crontab (no prompt)
sudo crontab -u <user> -e     # edit another user's crontab (root only)
```

## System-wide cron locations

```
/etc/crontab            system crontab; format has an extra USER field:
                         * * * * * root /path/to/script.sh
/etc/cron.d/*            drop-in files, same format as /etc/crontab (package-installed jobs)
/etc/cron.daily/         executable scripts, no schedule needed, run via run-parts
/etc/cron.hourly/
/etc/cron.weekly/
/etc/cron.monthly/
```

## Cron environment gotcha pattern (always do this)

```
# top of crontab: set PATH explicitly, cron's default is minimal
PATH=/usr/local/bin:/usr/bin:/bin

# always use absolute paths + redirect output, never rely on cron mail
0 2 * * * /usr/bin/python3 /opt/scripts/backup.py >> /var/log/backup.log 2>&1
```

## Anacron

```
/etc/anacrontab                system-wide anacron job definitions
/var/spool/anacron/             per-job "last ran" timestamp files

# /etc/anacrontab format:
# period(days)  delay(min)  job-id           command
7                25          weekly-backup    /opt/scripts/weekly-backup.sh
```
Period is in whole days (coarse) — not for minute-level scheduling. Designed to catch up jobs missed while the machine was powered off.
