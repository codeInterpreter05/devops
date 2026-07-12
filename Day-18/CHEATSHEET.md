# Day 18 — Cheatsheet: Storage & Filesystems

## Storage type quick reference

```
Block   raw addressable blocks, single-attach norm, lowest latency   e.g. EBS, /dev/sda
File    shared hierarchical namespace, POSIX-ish, multi-client       e.g. NFS, EFS
Object  flat namespace, HTTP API, no locking, whole-object writes    e.g. S3, Glacier
```

## LVM — physical / volume group / logical volume

```bash
# Physical Volumes
pvcreate /dev/sdb              # initialize a disk/partition as a PV
pvdisplay /dev/sdb              # detailed info
pvs                              # short list (device, VG, size, free)
pvremove /dev/sdb               # remove PV metadata (must not be in a VG)

# Volume Groups
vgcreate vg_data /dev/sdb        # create VG from one or more PVs
vgextend vg_data /dev/sdc        # add another PV's space to the pool
vgreduce vg_data /dev/sdc        # remove a PV from the VG (must be empty)
vgdisplay vg_data                # detailed info
vgs                              # short list (name, #PVs, #LVs, size, free)
vgremove vg_data                 # delete VG (must have no LVs)

# Logical Volumes
lvcreate -L 10G -n lv_data vg_data        # fixed-size LV
lvcreate -l 100%FREE -n lv_data vg_data   # use all remaining VG space
lvextend -L +5G /dev/vg_data/lv_data       # grow LV by 5G
lvextend -l +100%FREE /dev/vg_data/lv_data # grow LV to fill all free VG space
lvextend -r -L +5G /dev/vg_data/lv_data    # grow LV AND filesystem in one step
lvdisplay /dev/vg_data/lv_data             # detailed info
lvs                                        # short list
lvremove /dev/vg_data/lv_data              # delete LV (unmount first)

# Snapshots (copy-on-write)
lvcreate -L 1G -s -n lv_data_snap /dev/vg_data/lv_data   # create snapshot
mount -o ro /dev/vg_data/lv_data_snap /mnt/snap           # mount it read-only
lvremove /dev/vg_data/lv_data_snap                        # discard snapshot
```

## Resizing the filesystem after an LV resize

```bash
resize2fs /dev/vg_data/lv_data     # ext2/ext3/ext4 — works online, grow or shrink
xfs_growfs /mount/point            # XFS — takes the MOUNT PATH not the device; GROW ONLY, no shrink
```
**Growing the LV does not grow the filesystem — always follow with `resize2fs`/`xfs_growfs`, or use `lvextend -r`.**

## mount / umount / lsblk / findmnt

```bash
mount /dev/sdb1 /mnt/data                  # mount a block device
mount -t nfs4 server:/export /mnt/nfs      # mount NFS explicitly
mount -o remount,rw /                      # remount with new options
mount -a                                    # mount everything in /etc/fstab not yet mounted (test after editing!)
umount /mnt/data                           # clean unmount
umount -l /mnt/data                        # lazy unmount (detach now, free once idle)
umount -f /mnt/nfs                         # force unmount (hung NFS mount)

lsblk                                       # tree view of block devices + mountpoints
lsblk -f                                    # + filesystem type and UUID
findmnt                                     # query live mount table
findmnt /data                               # is this path currently a mountpoint?
fuser -mv /mnt/data                        # what's using a busy mount
lsof +D /mnt/data                          # open files under a path
```

## `/etc/fstab` field reference

```
<device>              <mountpoint>   <fstype>   <options>              <dump>  <pass>
UUID=xxxxxxxx-xxxx     /data          xfs        defaults,noatime        0       2
/dev/sdb1              /mnt/extra     ext4       defaults,nofail         0       2
server:/export         /mnt/nfs       nfs4       defaults,_netdev,nofail 0       0
tmpfs                  /tmp           tmpfs      defaults,size=2G        0       0
/swapfile              none           swap       sw                      0       0
```

| Field | Meaning |
|---|---|
| device | `UUID=...` (preferred, stable), `LABEL=...`, `/dev/sdX` (can shift), `server:/export` for NFS |
| mountpoint | directory to mount onto (`none` for swap) |
| fstype | `ext4`, `xfs`, `nfs4`, `vfat`, `tmpfs`, `swap`, `auto` |
| options | `defaults`, `noatime` (perf), `ro`/`rw`, `nofail` (don't block boot if missing), `_netdev` (wait for network) |
| dump | legacy `dump` backup flag — almost always `0` |
| pass | fsck order: `0` = skip, `1` = root, `2` = everything else |

```bash
blkid                     # list UUIDs/types for all block devices
blkid /dev/sdb1           # just one device
e2label /dev/sdb1 mylabel  # set an ext label
xfs_admin -L mylabel /dev/sdb1   # set an XFS label
```

## NFS

```bash
# server
cat /etc/exports          # /data 10.0.0.0/24(rw,sync,no_subtree_check)
exportfs -ra                # reload exports
exportfs -v                 # show current exports

# client
showmount -e nfsserver       # list what a server exports
mount -t nfs4 nfsserver:/data /mnt/data
mount -t nfs4 -o soft,timeo=30 nfsserver:/data /mnt/data   # fail fast instead of hanging forever
```
Common failure signatures: **"Stale file handle"** → unmount/remount, can't fix in place. Hung shell/`df` → likely a `hard`-mounted NFS share to an unreachable server. **"Permission denied"** as root on the client → `root_squash` mapping root to `nobody` server-side (expected security behavior).

## Disk space & inode usage

```bash
df -h                      # space usage per filesystem, human-readable
df -i                      # INODE usage — a "full disk" error can be inodes, not space
du -sh /var/log/*          # per-directory size breakdown
du -sh /var/log/* | sort -rh | head   # biggest directories first
lsof | grep deleted        # find space held by deleted-but-open files (df/du mismatch)
```

## Simulating a disk-full scenario

```bash
fallocate -l 1.8G /mnt/data/bigfile        # fast, allocates real blocks
dd if=/dev/zero of=/mnt/data/overflow bs=1M count=500   # fills disk, will error when space runs out
mkfs.ext4 -N 100 /dev/vg_data/lv_test       # force a filesystem with only 100 inodes (for inode-exhaustion tests)
```

## `iostat` / `iotop` — disk I/O diagnostics

```bash
iostat -xz 1               # extended stats, skip idle devices, 1s refresh
iostat -xz 5 3             # 5s interval, 3 samples then exit
```
| Column | Meaning |
|---|---|
| `%util` | % of time device was busy servicing I/O — sustained ~100% suggests saturation |
| `await` | avg I/O completion time (ms), including queueing — the real latency signal |
| `r/s` `w/s` | reads / writes per second (IOPS) |
| `rkB/s` `wkB/s` | throughput |
| `aqu-sz` (`avgqu-sz`) | average queue length — growing means device can't keep up |
| `svctm` | deprecated, don't rely on it |

```bash
iotop                       # live per-process DISK READ / DISK WRITE (needs root)
iotop -o                    # only show processes doing I/O right now
iotop -a                    # accumulated totals instead of per-second rate
```
Workflow: `df -h` (space) → `df -i` (inodes) → `lsblk -f` (which device) → `iostat -xz 1` (is the device saturated) → `iotop` (which process).
