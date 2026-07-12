# Day 3 — Cheatsheet: Processes & System

## Process states (`STAT` column)

```
R   running or runnable (on CPU or in the run queue)
S   interruptible sleep (waiting on an event, can be woken by a signal)
D   uninterruptible sleep (usually disk/NFS I/O — NOT killable, even with -9)
T   stopped (via SIGSTOP/SIGTSTP, or being traced)
Z   zombie (exited, exit status not yet reaped by parent — <defunct> in ps)
+   in the foreground process group
s   session leader
l   multi-threaded
<   higher priority        N   lower priority (niced)
```

## `ps` — snapshot

```bash
ps aux                                             # BSD style: all processes, all users
ps -ef                                              # SysV style: PPID shown, different columns
ps -ef --forest                                     # ASCII-art parent/child tree
ps -eo pid,ppid,stat,pcpu,pmem,cmd --sort=-pcpu     # custom columns, sorted by CPU desc
ps -eo pid,ppid,stat,cmd | awk '$3 ~ /Z/'           # find zombies
ps -o pgid= -p <PID>                                 # get the process group ID of a PID
pstree -p <PID>                                      # visualize tree with PIDs
```

## `top` / `htop`

```bash
top                     # launch; P = sort by CPU, M = sort by mem, k = kill, r = renice, 1 = per-core, q = quit
htop                    # friendlier: F5/t = tree view, F7/F8 = renice, F9 = kill w/ signal picker, / = search
```
Load average (top header, 3 numbers = 1/5/15 min): always read relative to `nproc` core count.

## `nice` / `renice` — CPU scheduling priority

```bash
nice -n 10 ./job.sh          # launch at niceness 10 (lower priority); range: -20 (highest) to 19 (lowest)
nice -n -5 ./job.sh           # launch at niceness -5 (higher priority) — requires root
renice -n 5 -p 1234            # change niceness of a running PID
renice -n -10 -p 1234           # requires root to lower niceness (raise priority)
ionice -c2 -n7 -p 1234           # I/O priority (separate mechanism from CPU nice)
```
Regular users may only raise their own process's niceness, never lower it or affect others' processes.

## Signals reference

```
1  SIGHUP   terminate (default) — reused by many daemons to mean "reload config"
2  SIGINT   terminate — sent by Ctrl-C
3  SIGQUIT  terminate + core dump — Ctrl-\
9  SIGKILL  terminate — cannot be caught, blocked, or ignored (kernel-enforced)
15 SIGTERM  terminate (default of plain `kill`) — catchable, graceful shutdown
18 SIGCONT  resume a stopped process
19 SIGSTOP  stop/pause — cannot be caught, blocked, or ignored
17 SIGCHLD  sent to parent on child exit/stop — triggers wait()/reaping
```

## `kill` / `killall` / `pkill` — sending signals

```bash
kill 1234                 # SIGTERM (default) to PID 1234
kill -15 1234               # explicit SIGTERM
kill -9 1234                # SIGKILL — force, no cleanup, last resort
kill -HUP 1234               # reload config (many daemons)
kill -STOP 1234              # pause
kill -CONT 1234               # resume
kill -l                      # list all signal names

kill -TERM -1234              # MINUS sign = target process GROUP 1234, not just PID 1234
kill -- -1234                  # same, defensively avoids flag-parsing ambiguity

killall nginx                  # signal by exact process NAME (all matching processes)
killall -9 nginx                 # force-kill by name

pkill -f "manage.py runserver"    # match full COMMAND LINE via regex
pkill -9 -u deploy                 # SIGKILL all processes owned by user deploy
```

## Job control

```bash
long_task.sh &         # run in background; shell prints job number + PID
jobs                    # list this shell's background/stopped jobs
jobs -p                 # list just the PIDs
fg %1                    # bring job 1 to foreground
bg %1                    # resume a stopped job in the background
Ctrl-Z                   # suspend current foreground job (SIGTSTP)
kill %1                   # signal by job number

nohup cmd > out.log 2>&1 &   # immunize against SIGHUP at launch; redirect since terminal will vanish
disown                        # detach most recent job from shell's job table (after the fact)
disown -h %1                   # keep in jobs list but exempt from SIGHUP-on-exit
disown -a                      # detach all jobs
```

## `lsof` — open files, sockets, deleted-but-held files

```bash
lsof -p 1234                  # every FD open by PID 1234
lsof -i :8080                  # what's bound to/connected on port 8080
lsof -i tcp -sTCP:LISTEN        # all TCP listeners
lsof /var/log/syslog             # who has this specific file open
lsof +D /var/log                  # recursive: everything open under a directory
lsof -u deploy                     # everything open by user deploy
lsof | grep deleted                 # files deleted but still held open (classic "disk full" cause)
```

## `strace` — syscall-level tracing

```bash
strace -p 1234                                  # attach live to a running process
strace ./prog arg1                               # launch under strace from the start
strace -f -p 1234                                 # follow forked children too
strace -e trace=open,openat,read -p 1234           # filter to specific syscalls
strace -c ./prog                                    # summary: count + time per syscall type
strace -o trace.log -p 1234                          # write to file instead of terminal
```
