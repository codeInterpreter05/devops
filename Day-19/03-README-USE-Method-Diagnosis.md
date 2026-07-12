# Day 19 — Performance & Debugging: The USE Method

**Phase:** 0 – Foundation | **Week:** W3 | **Domain:** Linux | **Flag:** ⚡ Interview-critical

## Brief

Anyone can run `top` and stare at it. What separates someone who can actually resolve a "the server is slow" page from someone who's guessing is a **systematic method** — a checklist that guarantees you cover every resource that could be the bottleneck, in an order that gets you to the answer fast, instead of randomly poking at whatever metric happens to be on screen. Brendan Gregg's **USE method** (Utilization, Saturation, Errors) is exactly that checklist, and it is *the* answer to this day's interview question: "walk me through your systematic diagnosis from CPU → memory → disk → network." Interviewers ask this specifically because most candidates either don't have a method at all (they free-associate: "I'd check `top`... and uh, logs...") or have only ever reacted to specific past incidents rather than internalized a transferable framework. This file is where that framework lives in depth.

## The method itself

For every system resource (CPU, memory, disk, network, and their sub-resources — per-core CPU, per-disk, per-NIC, etc.), check three things:

- **Utilization** — the percentage of time the resource was busy servicing work (or, for resources like memory, the percentage of capacity in use). High utilization means the resource has little headroom left.
- **Saturation** — the degree to which the resource has extra work it can't service right now, usually reflected as a queue. A resource can be at 100% utilization with *no* saturation (perfectly efficient, no backlog) or be saturated even below 100% utilization (bursty, queued work). Saturation is the more urgent signal — it means work is actively being delayed.
- **Errors** — the count of error events for that resource (retransmits, checksum failures, dropped packets, disk I/O errors, corrected/uncorrected memory errors). Errors can degrade performance independently of utilization/saturation looking fine, and they're the resource-health check people skip most often.

Gregg's insight: instead of trying to remember every tool and metric ad hoc, walk every resource through this same three-question checklist. It converts "I don't know what to check next" into "I haven't checked saturation on disk yet" — a concrete, actionable next step, every time.

## USE applied to CPU

- **Utilization**: `mpstat -P ALL 1` or `top`/`sar -u` — percentage of time each CPU core spent non-idle. Check *per-core*, not just the aggregate: one pegged core in an otherwise idle 32-core box (a single-threaded bottleneck) looks totally different from all cores evenly at 80%.
- **Saturation**: the run-queue length — `vmstat`'s `r` column (processes runnable and waiting for a core), or load average once normalized against core count (load average consistently above `nproc` means saturation). `sar -q` also reports run-queue size directly.
- **Errors**: less commonly surfaced for CPU on typical x86 servers, but includes things like machine-check exceptions (`mcelog`, hardware faults) — rare, but real, and worth a mental checkbox even if you skip actually checking it 99% of the time.

**Diagnosis pattern**: high utilization + low saturation = the CPU is busy but keeping up (maybe fine, maybe worth optimizing the hot code with `perf record`). High utilization + high saturation (`r` >> core count) = CPU-bound and actively falling behind — you need more cores, fewer processes competing, or faster code.

## USE applied to memory

- **Utilization**: percentage of RAM in use — from `free -h`'s `available` (inverse of free headroom) or `/proc/meminfo`'s `MemAvailable` vs `MemTotal`. Remember from file 1: high `used`/`buff/cache` alone isn't utilization trouble; the meaningful number is how much `available` is shrinking.
- **Saturation**: swapping activity — `vmstat`'s `si`/`so` columns being sustained and nonzero, or the kernel's OOM killer firing (`dmesg | grep -i "out of memory"`, or `oom_kill` entries in `/var/log/messages`/`journalctl -k`). Any active swapping under real load is memory saturation — the system has run out of physical RAM and is paging to a device orders of magnitude slower than RAM.
- **Errors**: allocation failures a process logs itself (`malloc` returning `NULL`, or in containerized environments, `cgroup` OOM kills that are distinct from system-wide OOM), and on servers with ECC RAM, correctable/uncorrectable memory errors (`edac-util`, `/sys/devices/system/edac/`) — a rising correctable-error count often precedes a hardware failure.

**Diagnosis pattern**: high utilization + zero swap activity = fine, cache is doing its job. High utilization + active `si`/`so` = saturated — the fix is more RAM, fewer/leaner processes, or fixing an actual leak, not "wait it out."

## USE applied to disk

- **Utilization**: `iostat -x 1` — the `%util` column, percentage of time the device was busy servicing I/O. Note this can mislead on modern SSDs/NVMe or RAID arrays that service multiple requests in parallel — 100% "busy" on such devices doesn't necessarily mean it's out of capacity, unlike a single spinning disk where it reliably does.
- **Saturation**: `iostat -x 1`'s `avgqu-sz` (average queue size) or `aqu-sz` on newer versions — requests queued waiting for the device. Also visible indirectly: `vmstat`'s `b` column (processes in uninterruptible sleep) climbing while `%iowait` in `sar -u`/`top` rises alongside it.
- **Errors**: `dmesg`/`journalctl -k` for I/O errors, SMART data (`smartctl -a /dev/sda`) for predictive failure indicators, and filesystem-level errors (remounts read-only, `EIO` returned to applications).

**Diagnosis pattern**: this is the pairing that resolves the load-average trap from file 1 — a high load average driven by processes in `D` state, paired with high `%util`/`avgqu-sz` on `iostat`, confirms the bottleneck is disk, not CPU, even though load average alone looked like a CPU problem.

## USE applied to network

- **Utilization**: throughput relative to link capacity — `sar -n DEV 1` or `ifstat`/`nload`, comparing observed Mbps/Gbps against the NIC's rated speed (`ethtool <iface>`).
- **Saturation**: dropped packets and queue backlog — `ip -s link show <iface>` (`RX-DRP`/`TX-DRP` counters), or `netstat -s`/`ss -s` showing retransmit counts and the TCP `Send-Q`/`Recv-Q` backlog for individual sockets. A network buffer overflowing shows up as drops, not a clean "100% utilized" number the way a CPU core does.
- **Errors**: interface error counters (`ip -s link show` — CRC errors, frame errors), TCP retransmit rate (`sar -n ETCP` / `nstat`), and connection resets. A rising retransmit rate at otherwise-low bandwidth utilization is the classic "the network *looks* fine by throughput but is actually degraded" signal — packet loss forcing TCP retransmission cripples throughput and latency without ever showing high raw utilization.

**Diagnosis pattern**: low utilization + high errors/retransmits = a flaky link, a bad cable/NIC, or an overloaded switch/router upstream, not a capacity problem — throwing more bandwidth at it won't help; you need to fix the actual loss source.

## Worked walkthrough: "a server is slow"

This is the structure to actually say out loud in an interview, resource by resource, fast:

1. **Frame it**: "First I'd confirm the symptom is real and get a time window — is it slow now, or was it slow at a specific past time? That decides whether I use live tools (`top`, `vmstat`) or historical ones (`sar` replaying `/var/log/sa/`)."

2. **CPU first** — cheapest to check, rules out or confirms the most common cause fast:
   ```bash
   nproc                    # know your core count before judging anything
   uptime                   # load average vs core count
   vmstat 1 5               # r (runnable) vs b (blocked) — disambiguates load average immediately
   mpstat -P ALL 1          # per-core utilization — catches single-threaded hot spots
   ```
   If `r` is high and per-core utilization is pegged: CPU-bound, go to `perf top`/`perf record` to find the hot function.

3. **Memory next** — rule out swapping, which makes everything else look slow as a side effect:
   ```bash
   free -h                  # check `available`, not `used`
   vmstat 1 5               # si/so columns — active swapping?
   dmesg | grep -i "kill"    # OOM killer activity
   ```
   If swap is active or OOM kills are in the log: this alone can explain broad, system-wide slowness (everything touching memory pays a swap-to-disk tax) — fix this before chasing anything else, since it can produce misleading downstream symptoms in disk/network checks too.

4. **Disk next** — if `b` (blocked processes) was elevated in step 2, this is where you confirm it:
   ```bash
   iostat -x 1 5             # %util and avgqu-sz per device
   sar -d 1 5                # historical equivalent
   ```
   High `%util`/queue size on a specific device, correlated with the `D`-state processes from step 2, confirms disk as the bottleneck — next step is finding *which* process/file with `iotop` or `strace -e trace=read,write,openat` on the offending PID (ties directly to file 2's technique).

5. **Network last** (often the least likely culprit for a single "slow server," more likely for "slow requests to a service"):
   ```bash
   ss -s                     # socket summary, retransmit counts
   sar -n DEV 1               # throughput per interface
   ip -s link show eth0        # error/drop counters
   ```
   Low throughput with rising retransmits/drops points at network loss rather than a capacity ceiling.

6. **Conclude with errors as a cross-cutting check**, not just a last resort: "At each resource I'd also check its error counters, not just utilization/saturation — a disk throwing `EIO`s or a NIC with rising CRC errors can degrade performance independent of how busy it looks, and that's the kind of thing utilization-only monitoring misses entirely."

The reason this ordering (CPU → memory → disk → network) works well as a default, and why the interview question is phrased in exactly that order: CPU and memory checks are near-instant and rule out (or strongly implicate) the two most common causes immediately; a memory problem (swapping) can produce misleading symptoms everywhere else, so it needs to be ruled out early; disk is usually the next most common bottleneck for application slowness (databases, log-heavy services); network is checked last because it's more often the cause of *distributed* slowness (a call to a dependency) than of a single box feeling generally sluggish. Say explicitly that this is a default order, not a rigid law — if the symptom description already points somewhere (e.g., "only slow on writes" screams disk), you start there instead.

## Points to Remember

- USE = Utilization, Saturation, Errors — walk every resource (CPU, memory, disk, network) through all three, not just utilization, which is the one most tools show by default and the one most people stop at.
- Saturation is the more urgent signal than utilization — a resource can be 100% utilized with no backlog (fine) or have a backlog even below 100% utilization (already a problem).
- Errors are the check people skip — a resource that looks fine on utilization/saturation can still be degrading performance via retries, retransmits, or I/O errors.
- The default CPU → memory → disk → network order exists because CPU/memory checks are fast and rule-out/rule-in quickly, and a memory problem can produce misleading symptoms in every layer checked afterward — but the order should bend toward whatever the reported symptom already hints at.
- `vmstat`'s `r`/`b` columns are the fastest way to route between "check CPU next" and "check disk next" after an ambiguous load-average reading.
- This method is a checklist, not a black box — the value in an interview is narrating *why* you check each thing, not just naming the tools.

## Common Mistakes

- Reciting tool names (`top`, `iostat`, `sar`) without the Utilization/Saturation/Errors structure behind them — interviewers are testing for the method, not a tool inventory.
- Stopping at utilization and never checking saturation or errors — declaring "disk is fine, only 60% utilized" while `avgqu-sz` shows a growing queue (already saturated at that utilization for that device).
- Treating 100% utilization the same on every device type — meaningful and urgent on a single spinning disk or a single CPU core; not automatically alarming on a RAID array, NVMe device, or multi-queue NIC that can still have headroom despite reporting "busy."
- Skipping errors entirely because they're the least glamorous check — missing a retransmit storm or a disk throwing `EIO`s because utilization and saturation both looked acceptable.
- Jumping straight to a deep tool (`perf record`, `strace`) before doing the cheap top-level USE pass — wasting time profiling code when the actual cause is system-wide swapping or a saturated disk that a 30-second `vmstat`/`free` check would have caught.
- Presenting the CPU → memory → disk → network order as a rigid rule rather than a sensible default — a good answer adapts the order to the symptom (e.g., "slow only on file uploads" should route straight to disk/network, not force a full CPU-first march through every resource).
