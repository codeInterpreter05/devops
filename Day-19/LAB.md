# Day 19 — Lab: Performance & Debugging

**Goal:** Go from "the server feels slow" to a specific, evidence-backed diagnosis using `strace`, `free`, `vmstat`, `sar`, `htop`, and (where available) `perf` — and practice the USE method on a live system.

**Prerequisites:** A Linux environment (Ubuntu via WSL2, a VM, or a cloud instance) with sudo access and Python 3 installed.

```bash
# Debian/Ubuntu
sudo apt update && sudo apt install -y sysstat strace ltrace htop \
    linux-tools-common linux-tools-generic python3

# RHEL/Fedora/Amazon Linux
sudo dnf install -y sysstat strace ltrace htop perf python3
```

`linux-tools-generic` (or a kernel-specific `linux-tools-$(uname -r)`) provides `perf` on Debian/Ubuntu — note that `perf` may still refuse to sample without elevated privileges on shared/cloud/container hosts (see Lab 5). That's expected and documented, not a setup failure.

---

### Lab 1 — Environment and tool inventory

1. Confirm every tool is installed and check versions:
   ```bash
   sar -V; strace -V; ltrace --version; htop --version; perf --version
   ```
2. Enable `sysstat`'s background data collection (the cron job that populates `/var/log/sa/`, which `sar` needs for historical replay):
   ```bash
   sudo sed -i 's/^ENABLED="false"/ENABLED="true"/' /etc/default/sysstat 2>/dev/null
   sudo systemctl enable --now sysstat
   systemctl status sysstat --no-pager
   ```
3. Confirm your core count and total memory before interpreting anything else today:
   ```bash
   nproc
   free -h
   ```

**Success criteria:** All five tools report a version with no "command not found," `sysstat`'s service is active, and you know your box's core count and total RAM by heart before starting Lab 2.

---

### Lab 2 — Core hands-on activity: profile a slow Python script with `strace`

This is the assigned hands-on activity for today — do it for real, not just read about it.

1. Set up a working directory and generate test data:
   ```bash
   mkdir -p ~/day19-lab && cd ~/day19-lab
   python3 -c "open('testfile.txt','w').write(('a'*79+'\n')*3000)"
   ls -la testfile.txt      # ~240 KB
   ```

2. Write the deliberately slow script — it disables Python's I/O buffering (`buffering=0`) and reads **one byte at a time**, so every single byte read is its own `read()` syscall:
   ```bash
   cat > slow_io.py << 'EOF'
   #!/usr/bin/env python3
   """Deliberately slow: unbuffered, byte-at-a-time file scanning."""
   import sys

   def count_lines(path):
       lines = 0
       with open(path, "rb", buffering=0) as f:
           while True:
               b = f.read(1)          # one syscall per byte
               if not b:
                   break
               if b == b"\n":
                   lines += 1
       return lines

   if __name__ == "__main__":
       print(count_lines(sys.argv[1]))
   EOF
   ```

3. Baseline timing — notice this is already slow in normal (non-traced) execution, because the per-byte syscall overhead is real, not a tracing artifact:
   ```bash
   time python3 slow_io.py testfile.txt
   ```

4. Get the syscall summary — this is the single command that diagnoses the bottleneck:
   ```bash
   strace -c -o strace_summary.txt python3 slow_io.py testfile.txt
   cat strace_summary.txt
   ```
   You should see `read` dominating both the call count and (usually) the time column, with a count in the tens of thousands — roughly one per byte in `testfile.txt`.

5. Drill into the actual calls to confirm the pattern (not just the count):
   ```bash
   strace -T -e trace=read -o strace_detail.txt python3 slow_io.py testfile.txt
   head -20 strace_detail.txt
   ```
   Look for lines like `read(3, "a", 1) = 1 <0.000004>` repeated — the literal signature of byte-at-a-time I/O.

6. Fix it — replace unbuffered single-byte reads with chunked reads through a real buffer:
   ```bash
   cat > fixed_io.py << 'EOF'
   #!/usr/bin/env python3
   """Fixed: reads in 64KB chunks instead of one byte at a time."""
   import sys

   def count_lines(path):
       lines = 0
       with open(path, "rb") as f:            # default buffering restored
           for chunk in iter(lambda: f.read(65536), b""):
               lines += chunk.count(b"\n")
       return lines

   if __name__ == "__main__":
       print(count_lines(sys.argv[1]))
   EOF
   ```

7. Re-measure both ways and compare:
   ```bash
   time python3 fixed_io.py testfile.txt
   strace -c -o strace_fixed_summary.txt python3 fixed_io.py testfile.txt
   cat strace_fixed_summary.txt
   ```

**Success criteria:** `strace_summary.txt` shows `read` with a call count in the thousands for `slow_io.py`; `strace_fixed_summary.txt` shows the `read` count dropping by two to three orders of magnitude for `fixed_io.py` (down to a handful of calls, one per 64KB chunk); wall-clock `time` output confirms the fixed version is measurably faster. You can explain, in one sentence, why the fix worked (fewer, larger syscalls instead of many tiny ones — the classic unbuffered-I/O bottleneck).

---

### Lab 3 — Reading `free`, `vmstat`, and `/proc/meminfo` under memory pressure

1. Capture a baseline:
   ```bash
   free -h
   vmstat 1 5
   grep -E "MemTotal|MemAvailable|Cached|Dirty" /proc/meminfo
   ```
2. Generate controlled memory pressure (adjust the size below to a fraction of your box's actual RAM — do not run this on a shared/production machine):
   ```bash
   python3 -c "
   import time
   block = bytearray(500 * 1024 * 1024)  # allocate ~500MB
   for i in range(len(block)):
       if i % 4096 == 0:
           block[i] = 1                  # touch each page so it's actually resident
   time.sleep(15)
   "
   ```
   Run this in one terminal; in a second terminal, watch `free -h` and `vmstat 1` every couple of seconds while it runs.
3. Note what changed: did `used` rise? Did `available` shrink proportionally? Did `buff/cache` shrink to make room (a sign the kernel reclaimed cache rather than swapping)? Check `si`/`so` in `vmstat` — did any actual swapping occur, or did the kernel absorb the pressure purely via cache reclaim?
4. Re-check `/proc/meminfo`'s `Cached` value before/after — a drop here (rather than swap activity) confirms the kernel freed page cache to satisfy the allocation, which is the healthy, expected outcome on a box with headroom.

**Success criteria:** You can point to the specific column (`available` in `free`, or `si`/`so` in `vmstat`) that would tell you *for certain* whether this pressure crossed from "benign cache reclaim" into "actual swapping," rather than just eyeballing that `used` went up.

---

### Lab 4 — Load average interpretation: CPU-bound vs. I/O-bound

1. Note your core count and current load average:
   ```bash
   nproc
   uptime
   ```
2. Generate pure CPU-bound load — one busy-loop process per core:
   ```bash
   for i in $(seq "$(nproc)"); do yes > /dev/null & done
   sleep 30 && uptime && vmstat 1 3 && pkill yes
   ```
   Expect load average to climb toward roughly your core count, `vmstat`'s `r` column to be high, and `b` to stay near 0.
3. Generate I/O-bound load instead — force synchronous, unbuffered writes so processes actually block waiting on the disk:
   ```bash
   for i in 1 2 3 4; do
     dd if=/dev/zero of=/tmp/dday19_$i.img bs=1M count=1000 oflag=dsync &
   done
   sleep 10 && uptime && vmstat 1 3
   wait
   ```
   `oflag=dsync` forces a synchronous flush after every write, so each `dd` spends real time in uninterruptible sleep waiting on the device rather than burning CPU.
4. Compare the two runs side by side: in the CPU-bound run, was `vmstat`'s `wa` (CPU % waiting on I/O) near zero? In the I/O-bound run, did `b` rise and `wa` rise together while `r` stayed low?

**Success criteria:** Using only `vmstat`'s `r`/`b` columns and `wa`, you can correctly state which of the two load-average spikes was CPU-driven and which was I/O-driven — without relying on the load average number alone, which looks similarly "high" in both cases.

---

### Lab 5 — `sar` history and live views with `htop`/`perf`

1. Confirm historical data exists (may take a few collection cycles after enabling `sysstat` in Lab 1 — if `/var/log/sa/` is empty, just use live mode for this lab):
   ```bash
   ls -la /var/log/sa/
   ```
2. Run `sar` live and, if a file exists for today, replay it:
   ```bash
   sar -u 1 5
   sar -q 1 5
   sar -u -f /var/log/sa/sa"$(date +%d)"
   ```
3. Launch `htop`, sort by CPU% (`F6`, or `>`/`<` depending on your version) and then by memory, and identify the top consumer of each.
4. Try `perf` against a busy process (reuse the `yes` loop from Lab 4 if needed for a target):
   ```bash
   yes > /dev/null & PID=$!
   perf stat -p "$PID" -- sleep 5
   kill "$PID"
   ```
   If you see a permission error, check the guardrail sysctl and note its value rather than changing it just for this lab:
   ```bash
   cat /proc/sys/kernel/perf_event_paranoid
   ```

**Success criteria:** You've seen both a live `sar` snapshot and (if history was available) a historical replay, can navigate `htop`'s per-resource sort views, and either got real `perf stat` output or correctly diagnosed *why* it was blocked via `perf_event_paranoid`.

---

### Lab 6 — Apply the USE method to your own machine

Walk every resource, right now, on your own box, filling in each cell:

| Resource | Utilization command | Saturation command | Errors command |
|---|---|---|---|
| CPU | `mpstat -P ALL 1 3` | `vmstat 1 3` (`r` vs `nproc`) | `dmesg \| grep -i mce` |
| Memory | `free -h` (`available`) | `vmstat 1 3` (`si`/`so`) | `dmesg \| grep -i "out of memory"` |
| Disk | `iostat -x 1 3` (`%util`) | `iostat -x 1 3` (`avgqu-sz`/`aqu-sz`) | `dmesg \| grep -iE "error|i/o"` |
| Network | `sar -n DEV 1 3` | `ss -s` (retransmits), `ip -s link show` | `ip -s link show` (CRC/drop counters) |

Run each command, record what you observe, and write one final verdict sentence: is this machine currently under resource pressure on any axis, and which one?

**Success criteria:** A filled-in USE table for CPU, memory, disk, and network on your own machine, plus a one-sentence verdict you could defend if an interviewer asked "so, is it fine?"

---

### Cleanup

```bash
pkill yes 2>/dev/null
rm -f /tmp/dday19_*.img
rm -rf ~/day19-lab
```

### Stretch challenge

Modify `slow_io.py` to reproduce a *different* bottleneck class: instead of reading one byte at a time, write a script that appends one line at a time to a file and calls `f.flush()` plus `os.fsync()` after every single line (simulating naive, unbatched logging). Use `strace -c` to confirm `write`/`fsync` dominate the call count this time instead of `read`. Then fix it by batching lines in memory and writing/flushing once at the end (or every N lines), and report the before/after syscall counts and wall-clock time.
