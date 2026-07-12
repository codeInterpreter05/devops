# Day 18 — Storage & Filesystems: AWS Storage & Disk I/O

**Phase:** 0 – Foundation | **Week:** W3 | **Domain:** Linux | **Flag:** —

## Brief

"When would you choose EBS over EFS?" is asked in almost every AWS-adjacent DevOps interview, because the honest answer requires understanding the block/file/object distinction from the first file in this day *and* applying it to real cost/performance tradeoffs — it's a fast filter for candidates who've actually provisioned storage versus those who've only clicked through the console once. The second half of this file — `iostat` and `iotop` — is the on-call skill that follows directly from that knowledge: when someone reports "the app is slow," disk I/O saturation is one of the first things to rule in or out, and these tools are how you do it without guessing.

## EBS vs. EFS vs. S3 (the interview-critical tradeoffs)

All three map onto the block/file/object model from the first file in this day, with AWS-specific mechanics layered on top.

**EBS (Elastic Block Store)** — block storage, attached to **one EC2 instance** in most cases (multi-attach exists only for `io1`/`io2` volumes paired with a cluster-aware filesystem — a narrow, uncommon case). An EBS volume is **AZ-scoped**: it can only attach to instances in the same Availability Zone it was created in, and moving it to another AZ requires snapshotting and restoring the snapshot as a new volume there.
- **Volume types:** `gp3` (general-purpose SSD, default choice today — baseline 3,000 IOPS / 125 MB/s independent of size, both provisionable higher for an extra cost, decoupled from capacity unlike its predecessor); `gp2` (older generation, IOPS scale with volume size and rely on a burst-credit model — a known gotcha where a small `gp2` volume runs out of burst credits under sustained load and throughput craters); `io1`/`io2` (provisioned IOPS, up to tens of thousands of IOPS, for the highest-performance database workloads, at a much higher cost per IOPS); `st1` (throughput-optimized HDD — cheap, high sequential throughput, poor at random I/O, good for big sequential workloads like log processing or big data); `sc1` (cold HDD, cheapest per GB, for rarely-accessed data).
- **Cost model:** you pay for **provisioned capacity** (GB-month) regardless of how much you actually use, plus provisioned IOPS/throughput if you've requested more than baseline. Oversizing an EBS volume "just in case" is pure wasted spend.
- **Use case:** anything that needs low-latency, high-IOPS block access from exactly one instance — database data directories, boot volumes, anything sensitive to I/O latency.

**EFS (Elastic File System)** — file storage, implemented as managed NFSv4.1 under the hood. It can be mounted concurrently by many EC2 instances across **multiple AZs** (and even on-premises over Direct Connect/VPN), with standard POSIX file semantics and locking.
- **Scaling:** no capacity to provision upfront — it grows and shrinks automatically as you add/remove data. Performance modes (General Purpose vs. Max I/O, the latter trading a bit of per-operation latency for much higher aggregate throughput/operations across many clients) and throughput modes (Bursting, Provisioned, Elastic) let you tune for the workload shape.
- **Cost model:** pay per GB actually stored (no waste from over-provisioning), with an Infrequent Access storage class for cost savings on cold data — but the per-GB price is meaningfully higher than EBS, and network-filesystem overhead means higher, less predictable per-operation latency than a local block device.
- **Use case:** shared state across a fleet — user-uploaded content served by many web servers, shared home directories, container/Lambda shared volumes, CI build caches shared across runners.

**S3 (Simple Storage Service)** — object storage: a flat namespace of objects (key + blob + metadata), accessed via HTTP REST API, not mounted as a POSIX filesystem by default.
- **Durability/scale:** designed for 11 nines of durability, virtually unlimited scale, storage classes (Standard, Standard-IA, One Zone-IA, Glacier Instant/Flexible/Deep Archive) trading retrieval speed for cost, lifecycle policies to auto-transition objects between classes as they age.
- **Consistency:** strong read-after-write consistency for all operations (since December 2020) — you no longer need to design around the older eventual-consistency caveats.
- **Cost model:** cheapest per GB by a wide margin, plus per-request pricing (PUT/GET/LIST costs add up at very high request rates) and data transfer costs out of the region.
- **Use case:** static assets, backups, logs, data lake storage, build artifacts — anything write-once/read-many that doesn't need file locking or in-place partial writes.

| Dimension | EBS | EFS | S3 |
|---|---|---|---|
| Storage model | Block | File (NFSv4.1) | Object |
| Attach scope | One instance (normally), one AZ | Many instances, many AZs | Unlimited, region-wide, via API |
| Access latency | Lowest | Medium (network fs overhead) | Highest per-request, scales horizontally |
| Provisioning | You provision size/IOPS upfront | Elastic, no upfront sizing | Elastic, no upfront sizing |
| Cost driver | Provisioned GB + IOPS, whether used or not | GB actually stored + throughput mode | GB stored (cheapest) + requests |
| Typical use | DB data dir, boot volume | Shared uploads, home dirs, fleet-wide state | Backups, static assets, data lake |
| Multi-writer | No (single-attach, normally) | Yes (POSIX locking) | Yes (no locking, whole-object overwrite) |

**Answering the interview question directly:** choose **EBS** when exactly one instance needs low-latency, high-IOPS block access and you can predict/control capacity — the classic case is a database's data directory or any boot volume. Choose **EFS** when *multiple* instances across possibly multiple AZs need concurrent, POSIX-correct access to the *same* file data — shared uploads, a CMS's media library, home directories mounted fleet-wide — and you're willing to trade a bit of per-operation latency and higher per-GB cost for that elasticity and not having to manage capacity or AZ placement. A common mistake in the other direction: reaching for EFS for a single-instance workload "to be safe" when a plain EBS volume would be both cheaper and faster, since EFS's network-filesystem overhead and per-GB price only pay off once you actually need multi-AZ, multi-writer access. And when the access pattern doesn't need file semantics at all (uploaded images being read back by URL, backups, logs), **S3** beats both — cheaper than either, and it removes the "which instance/AZ owns this volume" problem entirely.

## Disk I/O diagnostic tools

Diagnosing "is disk I/O the bottleneck" is a layered process: check space first (a full disk causes very different symptoms than a saturated one), then check device-level saturation, then find the specific process responsible.

**`lsblk`** — list block devices as a tree, showing how disks, partitions, and LVM logical volumes relate and where each is mounted:
```bash
lsblk                  # tree view: NAME, SIZE, TYPE, MOUNTPOINT
lsblk -f               # add FSTYPE and UUID columns
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,FSTYPE
```

**`df -h`** — filesystem-level space usage, human-readable:
```bash
df -h                  # usage per mounted filesystem
df -i                  # INODE usage, not space — a disk can show free space in `df -h`
                        # and still refuse to create files if inodes are exhausted
                        # (classic case: a directory full of millions of tiny files)
du -sh /var/log/*       # which directories are actually consuming the space df -h reports
```
`df` and `du` can disagree: a file that's been deleted but is still held open by a running process still consumes space (`df` reflects it) even though it no longer appears anywhere in the tree (`du` won't count it) — `lsof | grep deleted` finds these.

**`iostat`** (from the `sysstat` package — `apt install sysstat` / `yum install sysstat`) — per-device I/O statistics, the primary tool for spotting a saturated disk:
```bash
iostat -xz 1            # extended stats (-x), skip idle devices (-z), refresh every 1s
```
Key columns to read:
- **`%util`** — percentage of time the device was busy servicing I/O. Sustained values near 100% mean the device is saturated (though for devices with internal parallelism like SSDs/NVMe this can be less clear-cut than on spinning disks).
- **`await`** — average time (ms) for I/O requests to complete, *including* time queued. This is the real latency signal an application would feel — rising `await` with steady IOPS is the clearest sign of a device that can't keep up.
- **`r/s` / `w/s`** — reads and writes per second (IOPS).
- **`rkB/s` / `wkB/s`** — throughput in KB/s.
- **`aqu-sz`** (or `avgqu-sz` on older versions) — average queue length; a growing queue means requests are arriving faster than the device can drain them.
- **`svctm`** — deprecated on modern kernels/tools; don't rely on it, use `await` instead.

Interpretation in practice: high `%util` **and** climbing `await` together mean the device itself is the bottleneck. High `%util` with flat, low `await` on an SSD/NVMe can just mean it's busy but still keeping latency acceptable — don't treat `%util` alone as proof of a problem.

**`iotop`** — like `top`, but for I/O, broken down **per process** (requires root, since it reads kernel I/O accounting):
```bash
iotop                   # interactive, live view of DISK READ / DISK WRITE per process
iotop -o                # only show processes actually doing I/O right now
iotop -a                # accumulated (total) I/O since iotop started, instead of a per-second rate
```
Use `iostat` to confirm a device is saturated, then `iotop` to find *which process* is responsible — `iostat` alone can't tell you that.

**A practical bottleneck-hunting workflow:**
1. `df -h` — rule out "the disk is just full" (very different fix from a performance problem).
2. `df -i` — rule out inode exhaustion, which produces the same "No space left on device" error as a full disk but isn't fixed by deleting large files.
3. `lsblk -f` — confirm which physical/logical device backs the mount point you're worried about.
4. `iostat -xz 1` — watch `%util` and `await` on that device to confirm saturation.
5. `iotop` — identify the specific process driving the load.

## Points to Remember

- EBS = block, one instance, one AZ, pay for provisioned capacity regardless of use — best for databases and boot volumes.
- EFS = file (NFSv4.1), many instances, many AZs, pay per GB stored — best for shared state across a fleet.
- S3 = object, HTTP API, flat namespace, cheapest per GB, no file locking or in-place edits — best for backups, static assets, data lakes.
- `gp3` decouples IOPS/throughput from volume size (unlike `gp2`, which ties IOPS to size and relies on burst credits that can run out under sustained load).
- `df -h` shows space; `df -i` shows inodes — a disk can be "full" on inodes while `df -h` still reports free space.
- `iostat -xz 1`: watch `%util` (device busy-ness) and `await` (real latency including queueing) together — `await` is the more reliable saturation signal.
- `iostat` tells you a device is saturated; `iotop` tells you which process is causing it. Use them together.

## Common Mistakes

- Defaulting to EFS "to be safe" for a single-instance workload — paying more per GB and eating extra network-filesystem latency for multi-AZ sharing capability that's never used.
- Putting a database's data directory on EFS (or worse, an S3-backed mount) instead of EBS, then being surprised by poor write latency and locking behavior under real load.
- Provisioning `gp2` for a workload with sustained (not bursty) I/O and running out of burst credits, causing throughput to collapse exactly when load is highest — `gp3` avoids this by decoupling IOPS from size entirely.
- Diagnosing "No space left on device" by checking `df -h` only, missing that the actual cause is inode exhaustion (`df -i` at 100%) from millions of small files, not raw byte usage.
- Reading `%util` from `iostat` as the whole story and ignoring `await` — a busy-but-fast SSD can show high `%util` without actually being a bottleneck; sustained rising `await` is the more trustworthy signal of real saturation.
- Running `iostat` without `-x`, missing the extended columns (`await`, queue size) that actually explain *why* a device looks busy, not just that it does.
- Forgetting `iotop` requires root — running it as a normal user and getting a permission error or blank data instead of per-process figures.
