# Day 3 — Processes & System: Monitoring Tools

**Phase:** 0 – Foundation | **Week:** W1 | **Domain:** Linux | **Flag:**

## Brief

Understanding process states (file 1) is theory; these are the tools you'll actually run when a server or container is misbehaving. In real incidents you rarely get to reproduce a bug locally with a debugger attached — you get `ssh` (or `kubectl exec`) access and a live, degrading system, and you need to figure out *what's consuming what* fast. `ps`, `top`, `htop`, `lsof`, and `strace` are the tools that answer "what's running," "what's using CPU/memory right now," "what files/sockets does this process have open," and "what is this process actually doing at the syscall level" — in roughly that order of how deep you need to go.

## `ps` — point-in-time process snapshot

`ps` (process status) prints a **snapshot** — unlike `top`, it doesn't refresh live, which makes it ideal for scripting and piping.

```bash
ps aux                 # BSD-style: every process, every user, full detail
ps -ef                 # UNIX-style: similar info, different columns (STIME vs START, PPID shown)
ps -ef --forest         # show the parent/child tree using ASCII art indentation
ps -eo pid,ppid,stat,pcpu,pmem,cmd --sort=-pcpu   # custom columns, sorted by CPU descending
```

Key columns in `ps aux`: `USER`, `PID`, `%CPU`, `%MEM`, `VSZ` (virtual memory size), `RSS` (resident set size — actual physical RAM used), `STAT` (state, see file 1), `START`, `TIME` (cumulative CPU time consumed), `COMMAND`.

**`ps aux` vs `ps -ef` — why both exist:** `ps` predates POSIX standardization and grew two competing flag dialects (BSD and UNIX System V). `aux` is the BSD style (no dashes, historically); `-ef` is the POSIX/SysV style. Both work on Linux because `procps` implements both dialects — pick one and be fluent in it, but recognize the other on sight.

**Finding a process tree:** `ps -ef --forest` or the dedicated `pstree` (often needs installing: `apt install psmisc`) both visualize the PID → PPID hierarchy — essential before you kill anything with children, since killing a parent without its children can orphan them (see file 1) rather than clean them up.

## `top` — live, interactive overview

```bash
top                 # launch
```
Inside `top`:
- `P` — sort by %CPU (default); `M` — sort by %MEM
- `k` — kill a process (prompts for PID, then signal)
- `r` — renice a running process
- `1` — toggle per-core CPU breakdown (vs. one aggregate line)
- `q` — quit

The header block shows **load average** (1/5/15-minute exponentially weighted averages of the run-queue length — roughly "how many processes wanted a CPU, on average"), memory, and swap usage. A load average of `4.00` on a 4-core box means the CPUs are fully saturated on average; on an 8-core box, it means 50% headroom. **Load average always needs to be read relative to core count** — a bare number means nothing without knowing `nproc`.

## `htop` — the friendlier, more actionable `top`

```bash
sudo apt install htop      # Debian/Ubuntu
sudo yum install htop      # RHEL/CentOS/Amazon Linux
htop
```

`htop` adds: colorized, scrollable process list, a visual tree view (`F5` or `t`), mouse support, easy multi-select for killing several processes at once, and a much friendlier UI for `nice`/`renice` (`F7`/`F8` to bump priority) and killing specific signals (`F9` opens a signal picker instead of assuming `SIGTERM`). It's not more *powerful* than `top` underneath (both read `/proc`), just more discoverable — which is exactly why it's worth having installed everywhere you have shell access, though `top` is the one guaranteed to exist on a stripped-down box (e.g., minimal container images) when `htop` isn't installed.

## `nice` and `renice` — influencing scheduling priority

Linux's CPU scheduler (CFS — Completely Fair Scheduler) decides how much CPU time each runnable process gets. **Niceness** is a hint you give the scheduler about how *willing* a process is to yield CPU to others — it does not set a hard priority, it biases the scheduler's fairness calculation.

- Niceness ranges from **-20** (highest priority, least "nice" to others) to **+19** (lowest priority, most "nice" — yields readily).
- Default niceness for a new process is **0**.
- Lower niceness = gets more CPU time relative to other processes when there's contention; it has **no effect at all** if the CPU isn't contended (a lone process at nice +19 on an otherwise idle 8-core box runs exactly as fast as at nice -20).

```bash
nice -n 10 ./heavy_batch_job.sh     # start a new process at niceness 10 (lower priority)
nice -n -5 ./important.sh           # start at niceness -5 (higher priority) — requires root for negative values
renice -n 5 -p 1234                  # change the niceness of an already-running PID 1234
renice -n -10 -p 1234                # requires root — only root can *decrease* niceness (increase priority) or lower another user's process below where it started
```

**Why the root restriction exists:** if any user could freely lower their own process's niceness, everyone would set -20 for everything and the "fairness" hint becomes meaningless — an unprivileged-abuse vector. Regular users may only *raise* niceness (be nicer, lower priority) on their own processes, never lower it.

**Nice vs. I/O priority:** `nice`/`renice` only affect CPU scheduling. For I/O-bound contention (a process hammering disk), the equivalent tool is `ionice` (different mechanism, different scheduler class entirely) — don't expect `renice` to help a process that's disk-bound rather than CPU-bound.

## `lsof` — list open files (and sockets, and much more)

On Linux, "everything is a file" extends to network sockets, pipes, and devices — `lsof` (list open files) shows you all of them, per process.

```bash
lsof -p 1234                  # every file descriptor open by PID 1234
lsof -i :8080                 # which process is listening on / connected to port 8080
lsof -i tcp -sTCP:LISTEN      # every process listening on any TCP port
lsof /var/log/syslog          # every process that currently has this specific file open
lsof +D /var/log              # every open file under a directory (recursive)
lsof -u deploy                 # every open file owned by user "deploy"
```

**The classic production use case:** "Address already in use" when starting a service, or a disk showing 100% full via `df` but `du` can't find what's using the space. The latter happens when a process still holds a file descriptor open on a file that's been **deleted** — the space isn't reclaimed until the last file descriptor referencing it is closed. `lsof | grep deleted` finds exactly these phantom-space-consuming file handles; the fix is either stopping/restarting the offending process or (for log files specifically) using `logrotate`'s `copytruncate` so the descriptor is never invalidated.

## `strace` — trace the syscalls a process makes

`strace` intercepts every system call a process makes to the kernel — the lowest-level, most "ground truth" view of what a program is actually doing (opening files, reading sockets, forking, waiting).

```bash
strace -p 1234                     # attach to a running process and watch its syscalls live
strace ./myprogram arg1 arg2       # launch a program under strace from the start
strace -f -p 1234                  # follow child processes too (-f), essential for anything that forks
strace -e trace=open,openat,read -p 1234   # filter to specific syscalls (much less noisy)
strace -c ./myprogram              # summarize: count + time spent per syscall type, instead of a live stream
strace -o trace.log -p 1234        # write output to a file instead of the terminal
```

**When to reach for `strace` over everything else:** a process that's "hung" with no obvious CPU/memory symptom — `strace -p <pid>` immediately shows whether it's blocked on a `read()` from a socket that will never respond, endlessly `stat()`-ing a missing config file, or spinning on something else entirely. This is the tool that turns "it's stuck" into "it's stuck waiting on a `connect()` to 10.0.0.5:5432 that's timing out" — a completely different, actionable diagnosis. Attaching requires `ptrace` permissions (root, or the `CAP_SYS_PTRACE` capability, or being the process's own owner with appropriate kernel settings) — expect permission errors in hardened containers/Kubernetes pods unless the capability was explicitly granted.

## Points to Remember

- `ps` is a snapshot (good for scripting/piping); `top`/`htop` are live, refreshing views (good for interactive investigation).
- `ps aux` (BSD dialect) and `ps -ef` (SysV dialect) show mostly the same data with different column layouts — both work on Linux via `procps`.
- Load average must be read against core count (`nproc`) — a raw number alone tells you nothing about actual saturation.
- `nice`/`renice` only bias CPU scheduling fairness; they don't guarantee a fixed share and have zero effect without contention. `ionice` is the equivalent for disk I/O.
- Only root can lower a process's niceness (raise its priority) — a regular user can only make their own processes *nicer* (lower priority), never higher.
- `lsof` reveals open files, sockets, and deleted-but-still-held files — the standard tool for "disk full but `du` can't find it" and "what's listening on this port."
- `strace -p <pid>` (add `-f` for children) shows exactly what syscall a "stuck" process is blocked on — the fastest way to turn a vague hang into a concrete diagnosis.

## Common Mistakes

- Reading `top`'s load average as an absolute number without checking core count — calling `2.5` "high load" on a 32-core box, or missing that `2.5` is critical on a 2-core box.
- Expecting `renice` to fix an I/O-bound process — niceness only governs CPU scheduling; a disk-thrashing process needs `ionice`, not `renice`.
- Deleting a huge log file with `rm` to free disk space, then being confused that `df` still shows the space as used — the writing process still holds an open file descriptor to the now-unlinked inode; the space is only released when that descriptor closes (restart the process, or use `> /path/to/file` to truncate in place instead of `rm`).
- Running `strace` on a production process without `-e trace=...` filtering — the unfiltered output is enormous and can itself slow the target process down noticeably (ptrace overhead), which matters if you're tracing something latency-sensitive.
- Forgetting `-f` when `strace`-ing a process that forks (e.g., a shell script or a supervisor) — you'll only see the parent's syscalls and miss where the actual hang is happening in a child.
- Installing `htop` as the *only* tool relied on, then being stuck on a minimal container/base image that only has `/proc` and `top` (or neither) available.
