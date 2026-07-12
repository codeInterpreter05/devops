# Day 19 — Resources: Performance & Debugging

## Primary (assigned)

- **Brendan Gregg's *Systems Performance* (the USE method blog posts)** — brendangregg.com/usemethod.html is the free, canonical write-up of the USE method used throughout this day's notes, with worked checklists per resource type. The book itself (*Systems Performance: Enterprise and the Cloud*, 2nd ed.) is the deeper, paid companion if you want the full treatment — but the free blog posts alone cover everything needed for the interview question this day is built around.

## Deepen your understanding

- **brendangregg.com** — beyond the USE method page, his site hosts free write-ups on `perf` (perf-tools.md), flame graphs (the visualization he invented for exactly this kind of CPU profiling), and Linux performance tools posters/diagrams that map every tool to the resource/layer it observes. Worth bookmarking as an ongoing reference for this entire domain, not just today.
- **`man strace`, `man ltrace`, `man vmstat`, `man free`, `man sar`** — run these directly on your box; the man pages for `vmstat` and `sar` in particular document every column abbreviation (`si`, `so`, `wa`, `%steal`, etc.) precisely, which is worth reading once end-to-end rather than only picking up piecemeal from tutorials.
- **`man 5 proc`** (or `man proc`) — documents every field in `/proc/meminfo`, `/proc/stat`, and `/proc/<pid>/status` in detail; the authoritative source for what each `/proc` field actually means when `free`/`vmstat`'s summarized view isn't enough.
- **sysstat documentation** (github.com/sysstat/sysstat) — official docs for the `sar`/`mpstat`/`iostat` toolchain, including how the background collection cron job and `/var/log/sa/` historical data actually work.

## Reference / lookup

- **`strace` official repo and docs** (strace.io, github.com/strace/strace) — full flag reference beyond what's in the cheatsheet here, including `-e trace=%memory`, `-e signal=`, and injection/fault-simulation flags (`-e inject=`) useful for chaos-style testing once you're comfortable with basic tracing.
- **Julia Evans' zines** (jvns.ca) — has several short, illustrated zines specifically on `strace`, debugging, and how the kernel/syscalls work; excellent for building fast intuition alongside Gregg's more systematic material.

## Practice

- **Your own machine, right now** — the most direct practice available: run today's lab, then deliberately introduce a bottleneck (a slow script, a `dd`-driven disk-saturation loop) and diagnose it cold using only the USE method checklist, without looking at notes.
- **OverTheWire: Bandit / Narnia-adjacent wargames** (overthewire.org) — not performance-specific, but continues building the raw command-line fluency (piping, process inspection, `/proc` navigation) that makes tool output easier to read quickly under pressure.
