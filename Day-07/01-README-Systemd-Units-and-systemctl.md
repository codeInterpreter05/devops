# Day 7 — Systemd & Cron: Unit Files & systemctl

**Phase:** 0 – Foundation | **Week:** W1 | **Domain:** Shell | **Flag:**

## Brief

`systemd` is PID 1 on virtually every modern Linux distribution (Ubuntu, Debian, RHEL/CentOS/Fedora, Amazon Linux 2023, Arch) — it's the first process the kernel starts, the parent of every other process, and the thing responsible for bringing the whole machine from "kernel just booted" to "all your services are running." Every app you deploy directly on a VM (not in a container) almost certainly runs as a systemd service, and every SRE/DevOps interview eventually asks some version of "how do you make sure a process comes back up after it dies." This note covers what a unit actually is, the anatomy of a unit file, and the `systemctl` commands you use to control units day to day.

This day is split into three focused files:

1. **This file** — unit types (service/timer/target), unit file anatomy, dependency ordering, and `systemctl`.
2. **[02-README-journalctl-and-Logging.md](02-README-journalctl-and-Logging.md)** — how systemd's journal works and how to query it.
3. **[03-README-Cron-and-Anacron.md](03-README-Cron-and-Anacron.md)** — crontab syntax and anacron for machines that aren't always on.

## What a "unit" actually is

Everything systemd manages is a **unit** — a standardized description of a resource and how to manage its lifecycle. Units are plain text files (INI-like syntax) that live in `/etc/systemd/system/` (admin-created, highest precedence), `/run/systemd/system/` (runtime-generated, volatile), or `/usr/lib/systemd/system/` (package-installed defaults, lowest precedence). There are many unit types, but the three that matter here:

- **`.service`** — a long-running process or a one-shot command. This is what you write for "run my Python app."
- **`.timer`** — a schedule that activates a matching `.service` unit. This is systemd's answer to cron.
- **`.target`** — a synchronization point / grouping of other units, roughly analogous to old-style SysV runlevels. `multi-user.target` is the normal "system is booted and ready for logins" target; `graphical.target` adds a display manager on top. When you `enable` a service, you're usually telling systemd "start this when you reach `multi-user.target`."

## Anatomy of a `.service` unit file

```ini
[Unit]
Description=My Python worker
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 /opt/myapp/worker.py
Restart=on-failure
RestartSec=5
User=myapp
WorkingDirectory=/opt/myapp
Environment=PYTHONUNBUFFERED=1

[Install]
WantedBy=multi-user.target
```

Three sections, always in this order by convention:

- **`[Unit]`** — metadata and *ordering/dependency* directives. `Description` shows up in `systemctl status` and logs. `After=`/`Before=` control **start order only** — they say nothing about whether the other unit actually starts successfully. `Wants=`/`Requires=` control **dependency strength**: `Wants=` pulls in the target unit but keeps going even if it fails to start; `Requires=` will stop *this* unit from starting if the required unit fails. The classic beginner mistake is assuming `After=` implies a dependency — it doesn't; you almost always need `After=` *and* `Wants=`/`Requires=` together (e.g., `After=network-online.target` + `Wants=network-online.target`) if your service genuinely needs the network up first.
- **`[Service]`** — how to actually run and supervise the process. `Type=` tells systemd how to know the service has "started" (`simple` = the moment `ExecStart` forks, systemd considers it up; `forking` = for old daemons that fork and exit the parent, needs `PIDFile=`; `oneshot` = runs to completion, useful for setup scripts, often paired with `RemainAfterExit=yes`; `notify` = the process itself tells systemd it's ready via `sd_notify()`, the most accurate but requires app support). `ExecStart=` is the command; `ExecStartPre=`/`ExecStartPost=`/`ExecStop=` let you hook setup/teardown. `Restart=` and `RestartSec=` are the auto-restart policy (see below — this is the interview-critical part).
- **`[Install]`** — what happens when you run `systemctl enable`. `WantedBy=multi-user.target` means "when the system reaches multi-user.target at boot, start me too." Units without an `[Install]` section can be started manually but can't be `enable`d for boot.

## `Restart=` policies — the auto-restart mechanism

This is the single most interview-relevant fact in this whole topic: **systemd, not cron, not a supervisor script, is what you use to auto-restart a crashed service**, and it's controlled entirely by two directives in `[Service]`:

- `Restart=no` (default) — never restart automatically.
- `Restart=on-success` — restart only if the process exited cleanly (exit code 0 or via a clean signal) — unusual, mostly for units meant to loop between clean runs.
- `Restart=on-failure` — restart if the process exited with a **non-zero code**, was killed by a signal, or timed out. **This is the one you want for "restart on crash."**
- `Restart=on-abnormal` — restart on signal, timeout, or watchdog failure, but not on non-zero exit codes.
- `Restart=on-watchdog` — restart only if the service's watchdog timer (`WatchdogSec=`) expires (the process stopped pinging systemd to say it's alive).
- `Restart=on-abort` — restart only if the process was terminated by an uncaught signal (e.g., a segfault), not on a normal non-zero exit.
- `Restart=always` — restart regardless of exit reason, including a clean/intentional exit. Useful for services that should genuinely never stay down, but be careful: it will also restart a service you `kill` manually unless you use `systemctl stop` (which suspends restart logic while you're actively stopping it).

Supporting knobs you should know alongside `Restart=`:

- **`RestartSec=`** — how long to wait before each restart attempt (default 100ms; set it to something like `5` in production to avoid hammering a resource that's failing fast).
- **`StartLimitIntervalSec=`** and **`StartLimitBurst=`** — a circuit breaker: if the unit restarts more than `StartLimitBurst` times within `StartLimitIntervalSec`, systemd gives up and marks it failed instead of restart-looping forever. Without this, a service that crashes instantly on every start would restart-loop at machine speed forever.
- **`TimeoutStartSec=`** / **`TimeoutStopSec=`** — how long systemd waits for a start/stop to complete before considering it failed/forcing a `SIGKILL`.

## `systemctl` — the control surface

```bash
sudo systemctl start myapp        # start now (does not survive reboot)
sudo systemctl stop myapp         # stop now
sudo systemctl restart myapp      # stop then start
sudo systemctl reload myapp       # ask the app to reload config without restarting (if supported)
sudo systemctl enable myapp       # symlink into WantedBy target — starts on boot, but NOT now
sudo systemctl enable --now myapp # enable AND start in one command
sudo systemctl disable myapp      # remove boot symlink — does NOT stop it if currently running
systemctl status myapp            # current state, recent log lines, main PID, memory
systemctl is-active myapp         # just prints active/inactive (good for scripts)
systemctl is-enabled myapp        # just prints enabled/disabled
sudo systemctl daemon-reload      # re-read unit files after you edit one — REQUIRED or changes are ignored
```

`daemon-reload` trips up almost everyone at least once: editing `/etc/systemd/system/myapp.service` and running `systemctl restart myapp` will restart the process using the **old, cached** unit definition, because systemd parses unit files into an internal representation at load time and doesn't watch the filesystem for changes. You must `daemon-reload` first (or the unit will appear to ignore your edits).

`enable` vs `start` is the other classic confusion: `enable` only wires up the boot-time symlink in `/etc/systemd/system/<target>.wants/`; it does not start the unit right now. `start` runs it now but doesn't persist across reboot. You need both — hence `enable --now` as the one-liner people reach for once they know it exists.

## Points to Remember

- `.service` = a process to run/supervise; `.timer` = a schedule that triggers a matching `.service`; `.target` = a grouping/synchronization point (like a runlevel).
- `After=`/`Before=` only control **ordering**; `Wants=`/`Requires=` control **dependency strength**. You typically need both an ordering and a dependency directive together.
- `Restart=on-failure` is the standard "restart on crash" policy; `Restart=always` also restarts on clean exits; `Restart=no` is the default (no auto-restart at all).
- `StartLimitBurst=`/`StartLimitIntervalSec=` prevent infinite fast restart loops — without them a crash-looping process would consume CPU restarting at systemd's maximum rate forever.
- `systemctl daemon-reload` is mandatory after any manual edit to a unit file — systemd caches parsed unit definitions and does not hot-reload them from disk.
- `enable` (persists across boot) and `start` (runs now) are independent — `enable --now` does both.

## Common Mistakes

- Editing a unit file, running `systemctl restart`, and being confused why "nothing changed" — forgetting `sudo systemctl daemon-reload`.
- Setting `Restart=always` on a service that's supposed to run once and exit cleanly (e.g., a migration script), causing systemd to restart it in a loop forever.
- Assuming `After=network.target` means the network is actually usable — `network.target` only means the network *stack* is initialized, not that an IP/DNS is up; for that you need `After=network-online.target` plus `Wants=network-online.target` (and the `NetworkManager-wait-online`/`systemd-networkd-wait-online` service installed).
- Enabling a service (`systemctl enable`) and assuming it's now running — it isn't, until the next boot or an explicit `start`.
- Leaving `Restart=on-failure` with the default `RestartSec=100ms` on a service that fails instantly (e.g., missing dependency), causing rapid restart-looping that floods the journal and burns CPU until `StartLimitBurst` finally trips.
- Forgetting `WantedBy=multi-user.target` in `[Install]` and then being unable to `enable` the unit at all ("Unit has no installation config").
