# Day 19 — Cheatsheet: Performance & Debugging

## Load average — quick facts

```
uptime  ->  load average: 2.05, 1.32, 0.89
                            1min  5min  15min

- Decayed average of processes RUNNABLE + in UNINTERRUPTIBLE SLEEP (D state)
- NOT a percentage. NOT CPU-only. Meaningless without core count.
- nproc               # know this before judging any load number
- load > nproc  sustained   -> work is queuing (CPU or I/O)
- load ~ nproc              -> fully utilized, roughly keeping up
- load < nproc               -> headroom

Trend reading:
  1min > 5min > 15min   -> load is BUILDING right now
  1min < 5min < 15min   -> load is RECOVERING from a spike
  all three high & flat  -> SUSTAINED pressure, not transient
```

## `free` — memory at a glance

```bash
free -h                 # human-readable
free -h -s 2             # repeat every 2s
```
```
               total        used        free      shared  buff/cache   available
Mem:            15Gi       4.2Gi       1.1Gi       210Mi        10Gi        10Gi
Swap:          2.0Gi          0B       2.0Gi
```
```
used         -> actively allocated, NOT reclaimable            (high = often fine)
buff/cache   -> page cache/buffers, reclaimed instantly on demand (high = healthy, not a leak)
available    -> THE number to alert on: usable without swapping, accounts for reclaimable cache
free         -> literally untouched RAM (naive, undercounts usable memory)
swap used>0  -> only worrying if it's actively changing (see vmstat si/so), not just nonzero
```

## `vmstat` — the disambiguator

```bash
vmstat 1 5               # 1s interval, 5 samples
```
```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
```
```
r   -> runnable processes waiting for CPU     (high vs nproc = CPU-bound)
b   -> processes in uninterruptible sleep (D)  (high = I/O-bound / blocked)
si/so -> swap in/out per second                (nonzero+sustained = ACTIVE swapping, bad)
swpd -> swap space currently occupied          (nonzero alone = harmless if si/so are 0)
wa  -> % CPU time waiting on I/O                (rises with b = confirms disk bottleneck)
st  -> % CPU stolen by hypervisor               (cloud/VM only; nonzero = noisy-neighbor host issue)
```

## `/proc/meminfo` — ground truth fields

```bash
cat /proc/meminfo
grep -E "MemTotal|MemAvailable|Cached|Dirty|SwapFree|Slab" /proc/meminfo
```
```
MemAvailable -> same number free's "available" column is derived from
Dirty        -> modified pages not yet flushed to disk; stuck/climbing = slow/saturated block device
Slab         -> kernel data-structure memory (inode/dentry caches); unbounded growth = possible slab leak
SwapFree      -> cross-check against vmstat si/so for real swap pressure
```

## `perf` — CPU profiling

```bash
perf stat -p <pid> sleep 5          # aggregate HW counters over 5s
perf stat -e cycles,instructions,cache-misses -p <pid> sleep 5
perf top                            # live, top-like, system-wide hot functions
perf record -F 99 -p <pid> -g -- sleep 30   # sample @99Hz w/ call graph
perf report                         # interactive breakdown of perf.data
perf record -F 99 -a -g -- sleep 10          # system-wide sampling, all CPUs
```
```
IPC (instructions per cycle) < ~1.0 + high cache-miss %  -> memory-bound, not raw-CPU-bound
Needs CAP_SYS_ADMIN/CAP_PERFMON or relaxed kernel.perf_event_paranoid
cat /proc/sys/kernel/perf_event_paranoid   # check before assuming perf "doesn't work"
```

## `sar` — historical + live system activity (sysstat package)

```bash
sar -u 1 5             # CPU: live, 5 samples 1s apart
sar -u -f /var/log/sa/sa12    # CPU: replay the 12th of the month
sar -q 1 5              # load average + run-queue size
sar -r 1 5               # memory utilization over time
sar -d 1 5                # per-disk I/O
sar -n DEV 1 5             # per-interface network throughput
sar -n ETCP 1 5             # TCP retransmit stats
mpstat -P ALL 1 5           # per-CPU-core utilization (also sysstat)
iostat -x 1 5                # per-device %util, avgqu-sz/aqu-sz, await
```
```
%iowait high, %user/%system low  -> CPUs idle-but-blocked on I/O, not compute-bound
%steal > 0                        -> hypervisor starving your VM (cloud noisy-neighbor)
Historical data lives in /var/log/sa/saDD — requires sysstat's cron job enabled
```

## `htop` — quick live overview

```bash
htop
F6 / >          # sort by column (CPU%, MEM%, etc. depending on version)
F5              # tree view (parent/child processes)
F9              # kill selected process
u               # filter by user
```

## `strace` — syscall tracing

```bash
strace ls                          # trace a fresh command
strace -p <pid>                    # attach to running process
strace -o out.txt ls                # write to file
strace -c -p <pid>                   # SUMMARY: count/time per syscall (start here)
strace -f -p <pid>                    # follow forks/threads
strace -ff -o trace -p <pid>            # follow forks, one output file per pid
strace -e trace=open,read,write -p <pid>  # filter to specific syscalls
strace -e trace=network -p <pid>          # filter to network syscalls
strace -e trace=file -p <pid>              # filter to filesystem syscalls
strace -T -p <pid>                          # show per-call duration
strace -tt -p <pid>                          # wall-clock timestamps (microsecond)
strace -s 200 -p <pid>                        # widen truncated string args (default 32 chars)
```
```
Reading:  syscall(args) = return_value
          -1 ENOENT (...)   -> failed call, errno spelled out
          fd numbers persist across lines -> follow them to tie calls to one file/socket

Diagnostic signature for I/O bottleneck:
  strace -c  shows read/write dominating call COUNT -> byte-at-a-time / unbuffered I/O
  strace -e trace=read  shows read(fd, "x", 1) = 1 repeated -> smoking gun

Overhead: ptrace-based, can be 2x-20x slowdown.
  - Can mask/alter timing-sensitive bugs (races, timeouts) — don't fully trust under trace
  - For low-overhead tracing, look to perf trace / bpftrace (eBPF) instead
```

## `ltrace` — library-call tracing

```bash
ltrace ./myapp
ltrace -p <pid>
ltrace -c ./myapp                 # summary mode, like strace -c
ltrace -e malloc+free ./myapp     # filter to specific library calls
```
```
Traces dynamic library calls (malloc, strcpy, printf, ...) — NOT kernel syscalls
Use when: CPU-bound in userspace, strace shows little happening between syscalls
LIMITATION: cannot see into STATICALLY LINKED binaries (Go binaries, musl-static,
            -static builds, many distroless/container images) — hooks the dynamic
            linker's PLT/GOT, which doesn't exist there. Falls back to: use strace instead.
Higher overhead than strace (library calls >> syscalls in frequency)
```

## The USE method — checklist per resource

```
For EVERY resource, ask all three:
  UTILIZATION  -> % time busy servicing work (or % capacity used)
  SATURATION   -> extra work queued/waiting that can't be serviced right now
  ERRORS       -> error/retry/drop counts for that resource

CPU:
  U: mpstat -P ALL 1 / sar -u          S: vmstat 'r' vs nproc, sar -q      E: dmesg | grep mce
MEMORY:
  U: free -h 'available' / MemAvailable S: vmstat si/so, dmesg oom-kill    E: edac-util, cgroup OOM
DISK:
  U: iostat -x '%util'                  S: iostat 'avgqu-sz'/'aqu-sz'      E: dmesg, smartctl -a
NETWORK:
  U: sar -n DEV (throughput vs link)    S: ss -s retransmits, Send-Q/Recv-Q E: ip -s link (CRC/drops)

Diagnostic order default: CPU -> Memory -> Disk -> Network
  (fast to check first; memory pressure can mislead every later check; bend order to symptom)
```
