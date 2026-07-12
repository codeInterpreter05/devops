# Day 18 — Storage & Filesystems: Mounts, fstab & NFS

**Phase:** 0 – Foundation | **Week:** W3 | **Domain:** Linux | **Flag:** —

## Brief

`/etc/fstab` is one of the few files where a one-character typo can leave a production box unbootable — it's read early in boot, before you have a normal shell to fix it, so understanding its exact syntax (not just "it mounts stuff") is a real operational safety skill, not trivia. NFS is the other half of today's file: it's still everywhere — Kubernetes NFS-backed PersistentVolumes, shared build caches, legacy enterprise file shares — and its failure modes (a hung `df`, a "stale file handle" error) are confusing the first time you meet them and instantly recognizable after.

## Mount points and mounting mechanics

Mounting is the act of attaching a filesystem — whether it lives on a local block device, a network share, or is purely virtual (`tmpfs`, `proc`) — onto a directory in the single tree rooted at `/` (see Day 1's filesystem hierarchy note for the tree itself). The directory you mount onto is called the **mount point**; anything that was already in that directory becomes invisible (not deleted, just hidden underneath the new mount) until you unmount.

```bash
mount /dev/sdb1 /mnt/data                 # mount a block device onto a directory
mount -t nfs4 nfsserver:/export /mnt/nfs  # mount a network share, explicit fstype
mount -o remount,rw /                     # remount root read-write (e.g. after booting read-only for fsck)
umount /mnt/data                          # unmount cleanly
umount -l /mnt/data                       # lazy unmount: detach now, free resources once no longer busy
umount -f /mnt/nfs                        # force unmount — for a hung/unreachable NFS mount
```

Useful companions:
```bash
lsblk -f                 # tree view of block devices, filesystem type, and current mountpoint
findmnt                  # query the live mount table (tree view, filterable)
findmnt /data            # is this specific path currently a mountpoint, and with what options?
cat /proc/mounts         # authoritative live view of what's mounted right now (kernel-maintained)
```

If `umount` fails with **"target is busy"**, something still has an open file handle or working directory inside the mount:
```bash
fuser -mv /mnt/data       # list processes using the mount
lsof +D /mnt/data         # same idea, different tool — list open files under the path
```
Kill or `cd` those processes out first rather than reaching straight for `umount -f`, which can leave applications with dangling file descriptors.

## `/etc/fstab` in depth

`/etc/fstab` ("filesystem table") tells the system what to mount automatically at boot (and gives `mount -a` a shortcut so you don't have to type long `mount` commands by hand). Each line has **six whitespace-separated fields**:

| # | Field | Meaning | Example |
|---|---|---|---|
| 1 | Device | What to mount | `UUID=1a2b3c4d-...`, `/dev/sdb1`, `nfsserver:/export`, `LABEL=data` |
| 2 | Mount point | Where to mount it | `/data`, `/`, `none` (for swap) |
| 3 | Filesystem type | `ext4`, `xfs`, `nfs4`, `vfat`, `tmpfs`, `swap`, `auto` | |
| 4 | Options | Comma-separated mount options | `defaults`, `noatime`, `ro`, `nofail`, `_netdev` |
| 5 | Dump | Legacy backup flag for the `dump` utility | `0` (almost always, in practice) |
| 6 | Pass | `fsck` order at boot | `0` = never check, `1` = root filesystem, `2` = checked after root, others in parallel |

Example entries:
```
UUID=1a2b3c4d-5e6f-7890-abcd-ef1234567890  /        ext4    defaults          0 1
UUID=9f8e7d6c-5b4a-3210-fedc-ba0987654321  /data    xfs     defaults,noatime  0 2
nfsserver.internal:/export                 /mnt/nfs nfs4    defaults,_netdev,nofail  0 0
tmpfs                                       /tmp     tmpfs   defaults,size=2G  0 0
/swapfile                                   none     swap    sw                0 0
```

**Why UUID instead of `/dev/sdX`:** device names are assigned by enumeration order at boot and **can shift** — attach a new disk before an existing one in the boot sequence, and what was `/dev/sdb` can become `/dev/sdc`. On AWS specifically, device naming on newer instance types (NVMe-backed) doesn't even respect the name you requested at attach time (`/dev/sdf` might show up as `/dev/nvme1n1`). A UUID is generated when the filesystem is created and doesn't change, so it's the safe, portable reference. Find UUIDs with:
```bash
blkid                       # list UUIDs and types for all block devices
blkid /dev/sdb1              # just one device
lsblk -f                     # UUIDs alongside the device tree
```
`LABEL=` is an alternative if you set a filesystem label yourself (`e2label`, `xfs_admin -L`) and prefer a human-readable name over a UUID string.

**Key options worth knowing:**
- `defaults` = `rw,suid,dev,exec,auto,nouser,async` bundled together — the sane baseline.
- `noatime` — skip updating the file's last-access timestamp on every read; meaningful I/O performance win on busy filesystems that don't need access-time tracking.
- `nofail` — **don't halt boot** if this device isn't present or the mount fails. Critical for anything optional (an extra data disk, a network share) — without it, a missing device can drop the whole boot into an emergency shell.
- `_netdev` — tells the boot process this is a network filesystem, so it waits for networking to be up before attempting the mount. Missing this on an NFS/iSCSI entry is a classic cause of "mount ran before the network was ready and failed" at boot.
- `ro` / `rw` — read-only or read-write.
- `noauto` — list the entry in fstab (so `mount /data` shorthand works) without mounting it automatically at boot.

**Test before you reboot.** After editing `/etc/fstab`, never just reboot and hope:
```bash
sudo mount -a          # mounts everything in fstab not already mounted, using the file as-is
```
If `mount -a` succeeds cleanly, your edits are syntactically and practically valid. If it errors, fix it *now*, while you still have a working shell — a bad required (non-`nofail`) entry can otherwise leave the system dropping to an emergency/rescue shell (`systemd` on modern distros, or an initramfs prompt) on the next boot, with no normal login available until someone fixes it from single-user/rescue mode or an attached recovery volume.

## NFS and network mounts

NFS (Network File System) lets a server export a directory that client machines mount as if it were local, giving multiple hosts POSIX-like concurrent access to the same files.

**Server side** — export a directory via `/etc/exports`:
```
/data    10.0.0.0/24(rw,sync,no_subtree_check)
/backups 10.0.0.50(ro,sync,root_squash)
```
```bash
exportfs -ra            # reload exports after editing /etc/exports
exportfs -v             # show current exports and their options
```

**Client side** — mount the export:
```bash
showmount -e nfsserver              # discover what nfsserver is exporting
mount -t nfs4 nfsserver:/data /mnt/data
```

**NFSv3 vs NFSv4:** NFSv3 is stateless and historically depended on `rpcbind`/portmapper with dynamically assigned ports (painful through firewalls). NFSv4 is stateful, runs over a single well-known port (2049), has file locking built into the protocol itself (rather than the separate NLM side-protocol v3 needed), and supports Kerberos-based auth. Default to NFSv4 (`mount -t nfs4`) unless you have a specific legacy reason to use v3.

**Common failure modes:**
- **Stale file handle** — the client holds a reference to a file/export that no longer matches server-side state (the export was unmounted/remounted on the server, the underlying filesystem was recreated, or the file was deleted while a client had it open). Shows up as `Stale file handle` on any operation against that path. Fix: unmount and remount on the client (`umount -f /mnt/data && mount /mnt/data`); the client-side cached handle can't be repaired in place.
- **Hung mount / hung shell** — NFS mounts default to `hard` (retry forever, silently, until the server responds) rather than `soft` (give up after a timeout). This is deliberate — it protects against silent data loss on writes that would otherwise appear to fail mid-flight — but it means a `df`, `ls`, or even an unrelated shell that happens to `cd` through the mount can hang indefinitely if the NFS server is unreachable. Mounting with `soft,timeo=<n>` trades that safety for responsiveness; use it only for read-mostly, non-critical mounts.
- **Boot hangs** — an NFS entry in `/etc/fstab` without `_netdev` (and ideally `nofail`) can be attempted before the network is up, or block boot entirely if the server is down.
- **Permission denied for root** — NFS applies `root_squash` by default: the client's `root` user is mapped to an unprivileged user (`nobody`) on the server side, so operations that assume root can do anything (chown, certain writes) fail unexpectedly. This is a security feature, not a bug — `no_root_squash` disables it if you genuinely need root parity, at a real security cost.
- **Locking oddities across reboots** — NFS locking state can get out of sync if a client or server reboots while locks were held; expect occasional need to remount after either side restarts unexpectedly.

## Points to Remember

- Mounting attaches a filesystem onto an existing directory (the mount point); prior contents of that directory are hidden, not deleted, until unmount.
- `/etc/fstab` fields in order: device, mount point, fstype, options, dump, pass. `pass=1` for root, `pass=2` for others, `pass=0` to skip fsck.
- Prefer `UUID=` (or `LABEL=`) over `/dev/sdX` in fstab — device names can shift on reboot/reattachment; UUIDs don't.
- Always run `mount -a` after editing fstab, before rebooting, to catch mistakes while you still have a working shell.
- `nofail` prevents a missing/failed mount from blocking boot; `_netdev` tells the boot sequence to wait for networking before attempting a network mount — network filesystem entries need both.
- NFS defaults to `hard` mounts (retry forever) for data-safety reasons — this is why an unreachable NFS server can hang commands, not just NFS operations, on clients that have it mounted.
- "Stale file handle" means the client's cached reference to server-side state is invalid — remount, don't try to repair in place.

## Common Mistakes

- Adding a new disk or NFS mount to `/etc/fstab` without `nofail`, then rebooting — if the device isn't present or the network isn't up in time, the box drops to an emergency shell instead of finishing boot.
- Referencing `/dev/sdb1` in fstab instead of its `UUID=`, then having the device renumber after a reboot or disk reattachment (very common on cloud VMs after resizing or reattaching volumes) — the wrong device, or nothing, gets mounted where you expected.
- Forgetting `_netdev` on an NFS/iSCSI fstab entry, causing intermittent boot failures depending on how fast networking comes up relative to local mount attempts.
- Assuming an unresponsive `df -h` or hung terminal means the whole box is dead, when it's actually one process blocked on a `hard`-mounted NFS share whose server is down — check for network mounts before rebooting a "frozen" machine.
- Treating "stale file handle" as something you can fix by retrying the same operation — it requires an unmount/remount (or a server-side fix) because the client's cached handle itself is invalid.
- Deploying an NFS mount with `no_root_squash` "to make permissions easier" without recognizing the security tradeoff — a compromised client then has root-equivalent access to the export.
