# Day 19 — Performance & Debugging: CPU and Memory Metrics

**Phase:** 0 – Foundation | **Week:** W3 | **Domain:** Linux | **Flag:** ⚡ Interview-critical

## Brief

"The server is slow" is the single most common incident you'll be handed with zero other context, and how you respond to it is one of the fastest signals of real operational experience an interviewer can probe. Today is about building the toolkit and mental models to turn that vague complaint into a specific, evidence-backed diagnosis — starting with the two resources people reach for first (and most often misread): CPU and memory. Misreading `free -h`'s "used" column or citing a load average number without knowing your core count are two of the most common on-the-job (and interview) mistakes, and both are fixed by understanding what these tools actually measure, not just how to run them.

This day is split into three focused files:

1. **This file** — CPU profiling with `perf` and `sar`, load average explained properly, and memory diagnostics with `free`, `vmstat`, and `/proc/meminfo`.
2. **[02-README-Tracing-Syscalls-And-Libraries.md](02-README-Tracing-Syscalls-And-Libraries.md)** — `strace` for syscall tracing (in depth, ties directly to today's hands-on lab) and `ltrace` for library-call tracing.
3. **[03-README-USE-Method-Diagnosis.md](03-README-USE-Method-Diagnosis.md)** — the USE method (Utilization, Saturation, Errors) as a repeatable framework for systematic performance diagnosis, with a full worked "server is slow" walkthrough.

## CPU profiling: `perf`

`perf` is the Linux kernel's built-in profiler, hooking into hardware performance counters (CPU cycles, cache misses, branch mispredictions) and software tracepoints. It's the tool you reach for when `top` tells you a process is CPU-bound but you need to know *why* — which function, which line, which syscall.

```bash
perf stat -p <pid> sleep 5      # aggregate hardware counters for a process over 5s
perf top                        # live, top-like view of which functions are burning CPU cycles system-wide
perf record -F 99 -p <pid> -g -- sleep 30   # sample stack traces at 99 Hz for 30s, -g captures call graphs
perf report                     # interactive, sorted view of where time was spent (from the perf.data file record produced)
```

Sample `perf stat` output:

```
 Performance counter stats for process id '4213':

          4,821.34 msec task-clock                #    0.964 CPUs utilized
             12,441      context-switches          #    2.580 K/sec
                 89      cpu-migrations            #   18.462 /sec
             28,004      page-faults               #    5.809 K/sec
     14,203,881,220      cycles                    #    2.946 GHz
      9,982,004,112      instructions              #    0.70  insn per cycle
       1,982,004,331      cache-references
        612,004,221      cache-misses              #   30.87 % of all cache refs

       5.001234567 seconds time elapsed
```

`insn per cycle` (IPC) below ~1.0 with high cache-miss percentage points at a memory-bound workload (waiting on cache/RAM), not a raw CPU-bound one — the fix there is usually data-locality/algorithmic, not "add more cores." A `perf record` + `perf report` flame-graph-style breakdown is how you find the *specific function* burning the cycles instead of just knowing "the process is using CPU."

`perf` requires kernel support and often elevated privileges (`kernel.perf_event_paranoid` sysctl, or `CAP_SYS_ADMIN`/`CAP_PERFMON`) — expect to hit permission errors on shared cloud instances or hardened containers.

## CPU: `sar` for historical trends

`sar` (System Activity Reporter, part of `sysstat`) is the tool for "what was CPU doing at 3am when the alert fired" — unlike `top`/`perf` which show the live *present*, `sar` reads from historical data files collected by a cron-driven `sadc` process (typically every 10 minutes, `/var/log/sa/saDD`).

```bash
sar -u 1 5        # CPU utilization, 5 samples, 1 second apart (live mode)
sar -u -f /var/log/sa/sa12   # replay CPU stats recorded on the 12th of the month
sar -q 1 5        # load average + run queue length
```

Sample `sar -u` output:

```
Linux 5.15.0 (web-01)  07/12/2026   _x86_64_  (4 CPU)

12:00:01 PM     CPU     %user     %nice   %system   %iowait    %steal     %idle
12:00:02 PM     all      12.34      0.00      3.21      45.67      0.00     38.78
```

The columns matter more than the tool: `%iowait` at 45% while `%user`+`%system` are low tells you the CPUs aren't doing compute work — they're idle-but-blocked waiting on I/O to complete. `%steal` is the one specific to virtualized/cloud instances: time your VM *wanted* to run but the hypervisor gave the physical CPU to a noisy neighbor instead — a nonzero `%steal` on a "slow" cloud VM points at the host, not your code.

## Load average, properly explained

`uptime` / the first line of `top` reports three numbers: `load average: 2.05, 1.32, 0.89` — these are exponentially-decayed moving averages of the **number of processes in the run queue or in uninterruptible sleep**, sampled over the last **1, 5, and 15 minutes** respectively.

Two things people get wrong constantly:

**1. Load average is not a percentage and is meaningless without knowing your core count.** A load of `4.0` is fully saturated on a 4-core box but only 25%-loaded on a 16-core box. Always check `nproc` (or `/proc/cpuinfo`, or `lscpu`) before interpreting a load number. A commonly cited rule of thumb: sustained load below your core count means the box is generally keeping up; load consistently above it means work is queuing.

**2. Load average is not just CPU demand — it's runnable *or* blocked-in-uninterruptible-sleep processes.** A process waiting on disk I/O (state `D` in `ps`/`top`, "uninterruptible sleep") counts toward load average exactly the same as a process spinning on the CPU. This is the classic trap: you see `load average: 40` on an 8-core box, assume "CPU is the bottleneck," start hunting for a runaway compute process — and the actual cause is a stuck NFS mount or a disk with high latency, with dozens of processes parked in `D` state waiting on it. Always cross-check with `vmstat` (`r` vs `b` columns, below) or `ps aux | awk '$8=="D"'` before assuming load average implies CPU pressure.

The 1/5/15-minute trend itself is diagnostic: rising (1min > 5min > 15min) means load is building right now; falling (1min < 5min < 15min) means a spike is recovering. A flat, high number across all three means sustained, not transient, pressure.

## Memory: `free -h`

```bash
free -h
```

```
               total        used        free      shared  buff/cache   available
Mem:            15Gi       4.2Gi       1.1Gi       210Mi        10Gi        10Gi
Swap:          2.0Gi          0B       2.0Gi
```

This is the single most misread command in this whole domain. Column meanings:

- **`used`** — memory actively allocated by processes (RSS-ish, not counting reclaimable cache). This number climbing does **not** by itself indicate a problem.
- **`buff/cache`** — page cache (recently read/written file contents) and buffers. Linux aggressively uses "spare" RAM to cache disk data because unused RAM is wasted RAM — this is reclaimed instantly under memory pressure, not "wasted" memory.
- **`available`** — the modern, *correct* column to look at. It's an estimate of memory available for new applications **without swapping**, accounting for reclaimable cache. This is what monitoring/alerting should key off — not `free` (the naive free column undercounts, since it excludes reclaimable cache) and not `used` alone.
- **`free`** — literally untouched memory. A low `free` number with high `buff/cache` and high `available` is a perfectly healthy, well-utilized system — Linux is doing its job.

**Why "used looking high is often fine":** on a box that's been up for weeks, `buff/cache` naturally grows to fill available RAM as the kernel caches every file it reads. A newcomer sees `used: 14Gi / 15Gi total` and panics; the correct read is `available: 10Gi` — plenty of headroom, the kernel is just using otherwise-idle RAM productively. The actual danger signal is **`available` shrinking toward zero and swap usage climbing**, not a high `used` number in isolation.

## Memory: `vmstat`

```bash
vmstat 1 5     # 1-second interval, 5 samples
```

```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 2  1      0 112340  20544 9821200    0    0    45   210 1502 3305 12  3 82  3  0
 3  4      0  98220  20544 9834100    0    0  1820  5400 1611 3540  8  4  40 48  0
```

Key columns beyond memory: the **`procs`** columns are the ones that pair with load average — **`r`** is the number of processes currently runnable (competing for CPU right now), **`b`** is the number in uninterruptible sleep (blocked, usually on I/O). If `r` climbs well above your core count, you're CPU-bound; if `b` climbs, you're I/O-bound — this is the fastest way to disambiguate what an elevated load average actually means. **`si`/`so`** (swap in/out) should be at or near zero on a healthy system — nonzero and sustained means the box is actively swapping, a real red flag (as opposed to `swpd`, which just shows swap *space in use*, which can be nonzero and harmless if `si`/`so` are 0). **`wa`** (CPU % waiting on I/O) rising alongside `b` growth confirms an I/O bottleneck, matching what `sar -u`'s `%iowait` would show.

## `/proc/meminfo` — the ground truth

`free` and `vmstat` are both just formatted views over `/proc/meminfo`. Reading it directly gives you fields those tools don't surface:

```bash
cat /proc/meminfo
```

```
MemTotal:       16384000 kB
MemFree:         1126400 kB
MemAvailable:   10452000 kB
Buffers:          204800 kB
Cached:          9821200 kB
SwapCached:            0 kB
Dirty:             12040 kB
Writeback:             0 kB
SwapTotal:       2097148 kB
SwapFree:        2097148 kB
Slab:             421200 kB
```

`Dirty` — pages modified in cache but not yet flushed to disk — climbing steadily and not draining points at a slow or saturated block device (the kernel can't write back fast enough). `SwapFree` vs `SwapTotal` confirms actual swap pressure independent of `vmstat`'s momentary `si`/`so` sample. `Slab` is kernel-internal data-structure memory (inode/dentry caches, etc.) — normally reclaimable, but a slab leak (some buggy kernel modules/drivers cause this) shows up as `Slab` growing unbounded while process-level `used` memory looks flat, which is why "check `/proc/meminfo` directly" is the escalation path when `free`'s summary doesn't add up.

## Points to Remember

- Load average = number of runnable + uninterruptible-sleep (`D` state) processes, decayed over 1/5/15 minutes — not a percentage, meaningless without knowing core count (`nproc`).
- Rising 1<5<15 = building load; falling 1>5>15 = recovering; flat and high across all three = sustained pressure.
- `free`'s `available` column (not `free` or `used`) is the correct at-a-glance memory-health number — it already accounts for reclaimable cache.
- High `buff/cache` is Linux using idle RAM productively, not a leak; it's reclaimed instantly on demand.
- `vmstat`'s `r`/`b` columns disambiguate a high load average: `r` growing = CPU-bound, `b` growing = I/O-bound.
- `si`/`so` in `vmstat` near zero = healthy; sustained nonzero = actively swapping, a real problem (distinct from `swpd`/`SwapFree`, which just reflect swap space in use).
- `perf stat`'s IPC (instructions per cycle) and cache-miss rate distinguish a compute-bound workload from a memory-bound one — relevant before assuming "just add more CPU" fixes anything.
- `sar` reads historical data (`/var/log/sa/`) collected by cron — the only way to answer "what was the box doing at 3am" after the fact, unlike `top`/`vmstat` which are live-only.
- `%steal` in `sar -u` is unique to virtualized/cloud hosts — nonzero means the hypervisor is starving your VM of physical CPU, not your workload's fault.

## Common Mistakes

- Citing a load average number in isolation ("load is 8!") without checking core count — 8 is fine on a 16-core box, critical on a 4-core box.
- Assuming high load average always means CPU pressure, without checking `vmstat`'s `r` vs `b` or `ps` for `D`-state processes — a stuck disk or NFS mount can drive load sky-high with near-zero CPU usage.
- Panicking over `free -h`'s `used` column looking close to `total`, without checking `available` — this is usually the page cache doing exactly what it's designed to do.
- Treating `buff/cache` as memory that's "wasted" or unavailable to applications — it's reclaimed on demand in a page fault, essentially for free.
- Confusing `swpd` (bytes currently sitting in swap) with active swapping — a nonzero `swpd` value with `si`/`so` both at 0 usually just reflects something swapped out long ago and never needed since; it's not an active problem.
- Running `perf record` on a production box without `-F` (sample frequency) tuned down, generating enormous overhead and a multi-GB `perf.data` file on a busy system.
- Treating `sar`'s live mode (`sar -u 1 5`) as a substitute for historical analysis — if `sysstat`'s cron job isn't enabled, there's no `/var/log/sa/` history to replay after an incident, which you only discover when you need it most.
