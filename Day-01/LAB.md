# Day 1 — Lab: Linux Fundamentals I

**Goal:** Navigate a real Linux filesystem confidently and use `find` to answer real questions about a system, without looking anything up.

**Prerequisites:** A Linux environment — Ubuntu via WSL2, a VM, or a cloud instance. Install `tldr`: `sudo apt install tldr` (or `npm install -g tldr` / `pip install tldr`), then run `tldr --update` once.

---

### Lab 1 — Tour the filesystem hierarchy

1. From your home directory, run `cd /` then `ls -la`. Identify, out loud or in a notes file, what each top-level directory is for (`/etc`, `/var`, `/usr`, `/opt`, `/proc`, `/tmp`, `/home`, `/root`, `/boot`, `/dev`, `/sys`).
2. Run `df -h` and match each mounted filesystem to a path in the tree. Which mount is root (`/`) on?
3. Run `cat /proc/cpuinfo` and `cat /proc/meminfo`. Then run `ls -la /proc/cpuinfo` — note the file size (usually 0). Explain in your own words why a "0-byte" file returns pages of content when read.
4. Compare `/bin/ls` and `/usr/bin/ls` — on a modern distro, run `ls -la /bin` and see it's a symlink to `/usr/bin`. Explain what this tells you about the historical `/` vs `/usr` split.

**Success criteria:** You can explain, without checking notes, what lives in `/etc`, `/var/log`, `/var/lib`, `/tmp`, and `/proc` — and why `/proc` is different from the rest.

---

### Lab 2 — Absolute vs. relative paths, hands-on

1. `cd /var/log` then run `ls ./` and `ls /var/log` — confirm they return the same thing, but only one works from anywhere.
2. `cd /tmp`, create a nested structure: `mkdir -p sandbox/a/b/c`.
3. From `/tmp`, `cd sandbox/a/b/c`, then get back to `/tmp` two different ways: once with `cd ../../..` and once with `cd /tmp`. Then try `cd -` to toggle back to `sandbox/a/b/c`.
4. Write a one-line reason why a cron job using `rm -rf ./sandbox` would be dangerous compared to using the absolute path.

**Success criteria:** You can navigate using both absolute and relative paths without hesitation, and can explain in one sentence why scripts should prefer absolute paths or explicit `cd`.

---

### Lab 3 — The core hands-on activity: find all `.conf` files in `/etc`

This is the assigned hands-on activity for today — do it for real, not just read about it.

1. Find every `.conf` file under `/etc`:
   ```bash
   find /etc -name "*.conf"
   ```
2. Count them:
   ```bash
   find /etc -name "*.conf" | wc -l
   ```
3. Now find only `.conf` files that mention the word `log` inside them:
   ```bash
   find /etc -name "*.conf" -exec grep -l "log" {} \; 2>/dev/null
   ```
4. Do the same search, but using `xargs` instead of `-exec`, and compare:
   ```bash
   find /etc -name "*.conf" -print0 | xargs -0 grep -l "log" 2>/dev/null
   ```
5. Find the 5 most recently modified `.conf` files under `/etc`:
   ```bash
   find /etc -name "*.conf" -printf '%T@ %p\n' 2>/dev/null | sort -rn | head -5
   ```
6. Find any `.conf` files larger than 10KB:
   ```bash
   find /etc -name "*.conf" -size +10k
   ```

**Success criteria:** You have a working, memorized command for "find all files matching a pattern under a directory," and can extend it with `-exec`/`xargs` to act on the results.

---

### Lab 4 — `man` and `tldr` in practice

1. Run `tldr find` and `tldr tar` — note how much faster you understand the *common* usage versus reading the full man page cold.
2. Run `man find` and locate the `-mtime` and `-newer` options in the full manual — note how much denser it is.
3. Run `man -k partition` (equivalent to `apropos partition`) to see how man's keyword search works across all installed man pages.
4. Run `man 5 crontab` vs `man crontab` (which defaults to section 1) — confirm they show different content.

**Success criteria:** You know when to reach for `tldr` (fast recall) vs `man` (complete, authoritative reference), and can use `man -k` to discover commands you don't know the name of yet.

---

### Cleanup

```bash
rm -rf /tmp/sandbox
```

### Stretch challenge

Write a one-line `find` command that lists every file under `/etc` modified in the last 24 hours, sorted by modification time, most recent first — without using `-printf` (hint: combine `find`, `stat`, and `sort`).
