# Day 7 — Resources: Systemd & Cron

## Primary (assigned)

- **DigitalOcean: How To Use Systemctl to Manage Systemd Services and Units** — free, the assigned starting point for this day. Clear, hands-on walkthrough of `systemctl`, unit files, and enabling/starting services — matches directly onto today's core lab.

## Deepen your understanding

- **`man systemd.service`** and **`man systemd.unit`** — run these on any systemd box; the authoritative reference for every directive in `[Unit]`/`[Service]`/`[Install]`, including the full list of `Type=` and `Restart=` values. No internet required.
- **DigitalOcean: Understanding Systemd Units and Unit Files** — a companion piece to the primary resource that goes deeper into unit types (service/timer/target/socket) and dependency directives.
- **Red Hat: Working with systemd Unit Files** (access.redhat.com documentation) — a more ops/production-oriented take on the same material, useful for seeing RHEL-family conventions if you'll be working in that ecosystem.
- **crontab.guru** — an interactive crontab expression builder/explainer; paste any cron expression and get a plain-English description, or build one field by field. Excellent for double-checking a schedule before you ship it.

## Reference / lookup

- **`man 5 crontab`** — the crontab file format reference (fields, special strings, environment behavior) directly on any box; `man 8 cron` covers the daemon itself.
- **`man systemd.timer`** — full reference for `OnCalendar=`, `OnBootSec=`, `OnUnitActiveSec=`, and `Persistent=`.
- **Arch Wiki: systemd** — consistently one of the best-maintained, most detailed general systemd references on the web, even if you're not running Arch.

## Practice

- **KodeKloud / Linux Foundation sandbox labs (systemd & cron modules)** — guided hands-on environments if you want more repetitions beyond today's lab before moving on.
- Repeat today's lab on a fresh VM from scratch, from memory, without these notes open — the fastest way to confirm you actually retained the `systemctl`/`journalctl`/crontab syntax rather than just having followed steps once.
