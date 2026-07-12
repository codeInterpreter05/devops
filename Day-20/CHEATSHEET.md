# Day 20 — Cheatsheet: Phase 0 Review + Mock Interview

One page pulling together the highest-yield commands across the whole foundation phase, organized by diagnostic layer, plus the systematic order to run them in.

## Systematic diagnosis order (run top to bottom, stop early if you find the cause)

```
1. uptime / w / nproc         -> how bad, how long, how many cores (context for load average)
2. top / mpstat -P ALL 1      -> is it actually CPU-bound? (us/sy high) or waiting? (wa high)
3. free -h / vmstat 1 5       -> memory pressure, swap activity (si/so), blocked processes (b)
4. df -h / iostat -x 1 5      -> disk space, disk busy time (%util) and queued I/O (avgqu-sz/aqu-sz)
5. ss -tunlp / ping / dig / curl -v   -> sockets, reachability, DNS, actual HTTP-layer behavior
6. journalctl -u <svc> / logs  -> application-level evidence once system resources look clean
7. strace -p <pid> / lsof -p <pid>    -> exactly what a specific stuck process is blocked on
```

## Linux navigation & find (Day 1)

```bash
ls -lart                              # long, all, reverse-time sort (newest last)
cd -                                   # toggle to previous directory
find /etc -name "*.conf"               # find by name
find /var/log -mtime -1                # modified in last 1 day
find / -size +100M                     # larger than 100MB
find /tmp -mtime +7 -delete            # delete matches older than 7 days
find . -name "*.log" -exec grep -l ERROR {} \;   # act on every match
man 5 crontab                          # file-format section vs `man crontab` (command, section 1)
tldr <command>                          # fast example-driven reference
```

## Storage, LVM, and fstab (Day 18)

```bash
lsblk                                   # block devices and partitions, tree view
df -h                                    # mounted filesystem usage
du -sh /path/*  | sort -rh               # per-directory usage, largest first

pvs / vgs / lvs                          # physical volumes / volume groups / logical volumes
pvcreate /dev/sdb1                       # initialize a disk/partition for LVM
vgextend vg_data /dev/sdb1                # add a physical volume to an existing volume group
lvextend -L +10G /dev/vg_data/lv_data      # grow a logical volume
resize2fs /dev/vg_data/lv_data              # grow the ext4 filesystem to match (xfs_growfs for XFS)

mount -o loop image.img /mnt/dir          # mount a file-backed image (needs the `loop` option)
mount -a                                   # mount everything in /etc/fstab (safe way to test changes without rebooting)
umount /mnt/dir                            # unmount

# /etc/fstab fields: <device> <mount point> <fs type> <options> <dump> <pass>
# always back up before editing:
cp /etc/fstab /etc/fstab.bak
```

## Performance diagnosis (Day 19)

```bash
free -h                                  # memory: used / available / swap
vmstat 1 5                                # 5 samples, 1s apart: r (run queue), b (blocked), si/so (swap), wa (I/O wait)
top ; htop                                 # live view; load average, per-process %CPU/%MEM
ps -eo pid,ppid,pcpu,pmem,rss,cmd --sort=-pcpu   # snapshot, sorted by CPU
mpstat -P ALL 1                            # per-core CPU breakdown

iostat -x 1 5                               # per-device %util (busy time) and avgqu-sz/aqu-sz (queued I/O = saturation)
sar -u 1 5                                   # CPU utilization history
sar -r 1 5                                   # memory utilization history
sar -d 1 5                                   # disk activity history
sar -n DEV 1 5                                # network throughput history

lsof -p <pid>                                # every file/socket a process has open
lsof -i :8080                                 # what's listening on/connected to a port
strace -p <pid>                               # live syscall trace of a running process
strace -f -p <pid>                            # follow forked children too
strace -c ./prog                               # summarize syscall counts/time instead of streaming
dmesg | grep -i oom                             # check for OOM-killer activity
journalctl -k | grep -i oom                      # same, via systemd journal
```

## Basic networking debugging (standard skill, review-day addition)

```bash
ping -c 4 <host>                          # L3/ICMP reachability + rough latency (doesn't confirm the app works)
traceroute <host>                          # hop-by-hop path; last responding hop = prime suspect
tracepath <host>                            # traceroute alternative, no root required

ss -tunlp                                    # listening sockets (TCP+UDP), with process names
ss -t state established                       # active TCP connections
ss -ti                                         # TCP connections with internal stats (rtt, cwnd, retransmits)
netstat -s                                      # protocol-level stats incl. retransmits (legacy, ss preferred)

dig <host>                                       # DNS resolution + timing (dig +stats for detail)
nslookup <host>                                   # simpler DNS lookup, less detail than dig

curl -v <url>                                      # verbose: shows connect/TLS/request/response phases
curl -o /dev/null -s -w \
  "dns:%{time_namelookup} connect:%{time_connect} ttfb:%{time_starttransfer} total:%{time_total}\n" <url>
                                                     # break down exactly which phase of the request is slow
```

## Python & config review (Days 15–17 topics)

```python
# safe subprocess: list-form, timeout, check
subprocess.run(["df", "-h", "/"], capture_output=True, text=True, timeout=10, check=True)

# NEVER: subprocess.run(f"ping -c 1 {host}", shell=True)  -- shell injection if host is untrusted

# boto3: minimal IAM action for listing instances is ec2:DescribeInstances
boto3.client("ec2").describe_instances(
    Filters=[{"Name": "tag:Environment", "Values": ["prod"]},
             {"Name": "instance-state-name", "Values": ["running"]}]
)
```

```yaml
# YAML list vs scalar — a classic Jinja2 gotcha
servers:
  - 10.0.0.1     # correct: a list, iterates as one item per server
  - 10.0.0.2
# servers: "10.0.0.1, 10.0.0.2"   # WRONG shape: a string is iterable too, so {% for %} silently
#                                   loops character-by-character instead of raising an error
```
