# Day 3 — Resources: Processes & System

## Primary (assigned)

- **Linux Command (linuxcommand.org)** — free, the assigned starting point for this day. Its "Writing Shell Scripts" and command-reference sections cover job control (`&`, `jobs`, `fg`/`bg`) and process basics in the same practical, example-first style used throughout this course.

## Deepen your understanding

- **`man 7 signal`** — run this on any Linux box; the authoritative reference for every signal number, name, default action, and which ones can't be caught/blocked/ignored (`SIGKILL`, `SIGSTOP`). No internet required.
- **The Linux Programming Interface** by Michael Kerrisk — the chapters on process creation (`fork`/`exec`), process termination, and signals are the deepest, most precise treatment of exactly what happens at the kernel/syscall level for everything in today's notes. Worth the library/purchase if you want to go past "how" into "why the kernel does it this way."
- **Julia Evans — "ps, top and htop" and "strace" zines** (jvns.ca) — short, illustrated, excellent for building intuition about what these tools are actually doing under the hood, written specifically for engineers debugging real production issues.
- **Brendan Gregg's blog** (brendangregg.com) — the reference author for Linux performance/observability tooling; his process- and signal-related posts go well beyond the basics once you're comfortable with today's material.

## Reference / lookup

- `man ps`, `man top`, `man kill`, `man nice`, `man renice`, `man lsof`, `man strace` — your on-box references, always accurate for the exact version installed on that machine.
- **ss64.com** (ss64.com/bash) — fast, example-driven syntax reference for `kill`, `ps`, `nice`, and most other shell/coreutils commands — a good middle ground between `tldr` and full `man` pages.
- **explainshell.com** — paste any `ps`/`kill`/`find`-style command with flags and get every part explained inline.

## Practice

- **Killercoda / Katacoda-style Linux scenarios** (killercoda.com) — free, browser-based Linux sandboxes; search their catalog for process-management and troubleshooting scenarios to practice `ps`/`kill`/`strace` without needing your own VM.
- **OverTheWire: Bandit** (overthewire.org/wargames/bandit) — the same wargame referenced on Day 1; several later levels specifically require inspecting running processes and their open files (`ps`, `lsof`) to find the next password, directly exercising this day's tools.
