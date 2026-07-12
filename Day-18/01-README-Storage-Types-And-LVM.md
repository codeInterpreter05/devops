# Day 18 — Storage & Filesystems: Storage Types & LVM Basics

**Phase:** 0 – Foundation | **Week:** W3 | **Domain:** Linux | **Flag:** —

## Brief

Every provisioning decision you'll make as a DevOps engineer — "what EBS volume type for this database," "should this shared upload directory be EFS or just a bigger disk," "how do I grow this volume group without taking the app down" — comes back to understanding what kind of storage you're actually looking at and how Linux stitches raw disks into usable filesystems. Day 1 covered the Filesystem Hierarchy Standard (what lives where once you're inside the tree); today goes one level down — the storage *underneath* that tree, and the abstraction layer (LVM) that most production Linux systems use instead of raw partitions.

This day is split into three focused files:

1. **This file** — block vs. object vs. file storage, and LVM basics (physical volumes, volume groups, logical volumes, resizing, snapshots).
2. **[02-README-Mounts-Fstab-And-NFS.md](02-README-Mounts-Fstab-And-NFS.md)** — `/etc/fstab` field-by-field, UUID vs. device path, `mount`/`umount`, and NFS/network mounts including failure modes.
3. **[03-README-AWS-Storage-And-Disk-IO.md](03-README-AWS-Storage-And-Disk-IO.md)** — EBS vs. EFS vs. S3 tradeoffs (the interview-critical content for today) plus disk I/O diagnostics: `iostat`, `iotop`, `lsblk`, `df -h`.

## Block vs. object vs. file storage

These are three fundamentally different models for how data is stored and accessed — mixing them up (e.g., trying to run a database directly on object storage) is a common junior mistake that causes real outages.

**Block storage** presents a raw device — a fixed-size array of addressable blocks, with no inherent structure. It doesn't know what a "file" is; that's the job of whatever sits on top of it. You (or the OS) partition it and put a filesystem on it (`mkfs.ext4`, `mkfs.xfs`), or a database engine can address it directly without a filesystem at all (Oracle ASM, some high-performance setups) for lower overhead.
- **Examples:** a raw disk (`/dev/sda`), an AWS EBS volume, a SAN LUN, an NVMe drive.
- **Access pattern:** the OS talks to it in fixed-size blocks (typically 4KB) via a block device driver — low latency, high IOPS potential, but normally attached to **one host at a time** (multi-attach is the exception, requires a cluster-aware filesystem, and is rare in practice).
- **Use case:** anything needing predictable, low-latency random I/O — a database's data directory, a VM's boot disk.

**File storage** exposes a hierarchical namespace (directories and files) over a network, with POSIX-like semantics (open/read/write/close, permissions, often file locking). A server manages the actual disks and filesystem; clients talk to it over a protocol.
- **Examples:** NFS, SMB/CIFS, AWS EFS.
- **Access pattern:** multiple clients can mount and access the **same** namespace concurrently — this is the key difference from block storage. You get shared access, at the cost of network latency and protocol overhead compared to a local block device.
- **Use case:** shared config directories, user home directories served to many hosts, application upload directories shared across a fleet of web servers.

**Object storage** stores whole "objects" (arbitrary blobs of data plus metadata) in a **flat namespace**, addressed by a key rather than a filesystem path — S3's `/` in an object key is a display convention in the console, not a real directory. Access is via an HTTP API (`PUT`, `GET`, `DELETE`, `LIST`), not POSIX file calls.
- **Examples:** AWS S3, Azure Blob Storage, Google Cloud Storage.
- **Access pattern:** no partial in-place edits — updating an object means overwriting the whole thing. No native file locking. Virtually unlimited scale and very high durability, but higher and less predictable latency per request than block storage, and not addressable as a normal filesystem by default (tools like `s3fs` or `goofys` fake a mountable filesystem over S3's API, but it's an approximation — things like `flock()`, atomic renames, and small random writes don't behave like a real filesystem, which is why databases should never live on an S3-backed mount).
- **Use case:** static assets, backups, logs, data lake storage, anything write-once/read-many.

| | Block | File | Object |
|---|---|---|---|
| Unit of access | Fixed-size blocks | Files/directories (POSIX-ish) | Whole objects (key + blob) |
| Protocol | Block device driver (local/iSCSI/EBS) | NFS, SMB | HTTP REST API |
| Concurrent multi-host access | Rare (single-attach is the norm) | Native (that's the point) | Native (many clients, HTTP) |
| Latency | Lowest | Medium (network + protocol) | Highest, but scales further |
| Typical example | EBS volume, local SSD | NFS share, EFS | S3, Glacier |
| Good fit | Databases, boot volumes | Shared home dirs, CMS uploads | Backups, static assets, data lake |
| Bad fit | Sharing across many hosts | Ultra-low-latency random I/O | Databases, anything needing locking |

## LVM basics

**Why LVM exists:** a plain disk partition (`/dev/sda1`) is a fixed-size, fixed-position slice of a disk. Growing it means repartitioning — historically requiring unmounting, sometimes booting from a rescue disk, and real risk of data loss. LVM (Logical Volume Manager) inserts an abstraction layer between raw disks and filesystems so you can add capacity, resize, and snapshot **without unmounting or taking downtime**, which is why it's the default disk layout on most RHEL/CentOS/Fedora installs and common on Debian/Ubuntu servers too.

LVM has three layers:

1. **Physical Volume (PV)** — a raw disk or partition initialized for LVM use. LVM writes metadata to it but the underlying block device is still a disk/partition/EBS volume.
   ```bash
   pvcreate /dev/xvdf          # initialize a whole disk as a PV
   pvs                         # list PVs (short form)
   pvdisplay /dev/xvdf         # detailed view
   ```
2. **Volume Group (VG)** — a storage pool made by aggregating one or more PVs. This is where the "no repartitioning" magic comes from: a VG can span multiple physical disks, and you grow it by adding another PV.
   ```bash
   vgcreate vg_data /dev/xvdf              # create a VG from one PV
   vgextend vg_data /dev/xvdg              # add a second disk to the pool later
   vgs                                     # list VGs, see total/free space
   ```
3. **Logical Volume (LV)** — a virtual "partition" carved out of a VG's free space. This is what you actually format with a filesystem and mount — functionally it behaves like `/dev/sda1` did, but it can be resized on the fly.
   ```bash
   lvcreate -L 10G -n lv_data vg_data       # fixed 10G LV
   lvcreate -l 100%FREE -n lv_data vg_data  # use all remaining VG space
   mkfs.ext4 /dev/vg_data/lv_data           # format it
   mount /dev/vg_data/lv_data /data         # mount it like any other block device
   lvs                                      # list LVs
   ```

**Resizing online (the core reason LVM exists):** attach a new disk, add it to the VG, grow the LV, then grow the filesystem *inside* the LV — all without unmounting:
```bash
pvcreate /dev/xvdg                          # 1. prep the new disk
vgextend vg_data /dev/xvdg                   # 2. add its space to the pool
lvextend -L +20G /dev/vg_data/lv_data        # 3. grow the logical volume by 20G
resize2fs /dev/vg_data/lv_data               # 4a. grow an ext4 filesystem to fill it
# xfs_growfs /data                           # 4b. grow an XFS filesystem instead (mounted path, not device)
```
Step 4 is the one people forget — **growing the LV does not grow the filesystem sitting inside it.** `df -h` will keep reporting the old size until you run `resize2fs` (ext4/ext3, works online) or `xfs_growfs` (XFS, always online — but note XFS can only grow, never shrink, by design). `lvextend -r` combines steps 3 and 4 automatically for supported filesystems and is the safer habit to build.

**Snapshots** use copy-on-write: a snapshot LV only stores blocks as they change on the origin, making it cheap to create and ideal for getting a consistent, point-in-time backup of a live volume without stopping the application:
```bash
lvcreate -L 2G -s -n snap_data /dev/vg_data/lv_data   # 2G reserved for changed blocks
mount -o ro /dev/vg_data/snap_data /mnt/snap          # mount read-only, back it up
lvremove /dev/vg_data/snap_data                       # discard when done
```
If the snapshot's reserved space fills up before you remove it (because the origin volume changed more than you allocated for), the snapshot becomes invalid and is dropped automatically — size snapshots generously for how long you intend to keep them and how write-heavy the origin volume is.

## Points to Remember

- Block = raw addressable blocks, one host at a time, lowest latency (EBS, local disk). File = shared hierarchical namespace over a network protocol (NFS, EFS). Object = flat namespace, HTTP API, no partial writes or locking (S3).
- Never put a database's data directory on object storage or an `s3fs`-style mount — no real file locking, no efficient small random writes, and it will corrupt or perform terribly under real workloads.
- LVM stack: Physical Volume → Volume Group → Logical Volume. The VG is the pool; the LV is what actually gets formatted and mounted.
- Extending an LV (`lvextend`) does **not** extend the filesystem inside it — you must also run `resize2fs` (ext4) or `xfs_growfs` (XFS), or use `lvextend -r` to do both in one step.
- XFS can only grow, never shrink. ext4 can technically shrink (`resize2fs` smaller, offline, unmounted, with `e2fsck` first) but it's rare and riskier — plan capacity growth-only where you can.
- LVM snapshots are copy-on-write and need their own reserved space sized for the amount of change expected — an undersized snapshot can silently become invalid.

## Common Mistakes

- Extending the logical volume with `lvextend` and stopping there — `df -h` still shows the old size because the filesystem itself was never grown. Always follow with `resize2fs`/`xfs_growfs`, or use `lvextend -r` / `lvresize -r`.
- Running `resize2fs` with a device path that doesn't match reality after LVM renames (using `/dev/sdb1` instead of the LVM device-mapper path `/dev/vg_data/lv_data` or `/dev/mapper/vg_data-lv_data`) — always target the LV, not the underlying PV.
- Treating object storage (S3, or an `s3fs` mount) as a POSIX filesystem for an application that needs file locking or frequent small in-place writes — it will work in a demo and fail under real concurrent load.
- Creating a VG from a single disk with no headroom, then being unable to extend it later because there's no spare disk to add and no unallocated space in the existing PV.
- Forgetting to size an LVM snapshot for the amount of write activity expected during the backup window — an exhausted snapshot silently becomes unusable, and you find out only when you try to restore from it.
- Deleting a PV's disk (or detaching an EBS volume) that's still part of an active VG without first checking `pvs`/`vgs` — this can corrupt the entire volume group, not just the one disk.
