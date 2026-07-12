# Day 18 — Lab: Storage & Filesystems

**Goal:** Build a real LVM stack from raw disks, simulate a disk-full scenario and diagnose it correctly, extend a live logical volume online with zero downtime, and use `iostat`/`iotop` to watch real disk I/O while it happens.

**Prerequisites:** The LVM labs need **at least one extra virtual disk beyond your root disk** — you can't safely practice this on a system's only disk.
- **VirtualBox:** shut the VM down, open *Settings → Storage*, add a new virtual hard disk (e.g. 5–10GB, VDI, dynamically allocated) to the existing SATA/VirtIO controller, then boot. It'll show up as a new device (commonly `/dev/sdb`).
- **Cloud VM (AWS EC2, or equivalent):** create and attach an extra EBS volume: `aws ec2 create-volume --availability-zone <az> --size 5 --volume-type gp3`, then `aws ec2 attach-volume --volume-id <vol-id> --instance-id <id> --device /dev/sdf` (device name may show up as `/dev/nvme1n1` on Nitro instance types — check with `lsblk`). Any equivalent (a second attached disk on GCP/Azure, or a second local VM disk) works the same way.
- Install `sysstat` for `iostat`, and `iotop` if not already present: `sudo apt install sysstat iotop` (Debian/Ubuntu) or `sudo yum install sysstat iotop` (RHEL/Amazon Linux).
- **Use a throwaway VM/instance for this lab** — several exercises intentionally fill a disk and corrupt fstab-adjacent state to practice recovery.

---

### Lab 1 — Inventory what you're starting with

1. Before touching anything, run `lsblk`, `df -h`, and `blkid` and record the output. Identify your root disk/partition and confirm the new disk you attached is visible (and has no partitions/filesystem on it yet — it should show as e.g. `sdb` with no children).
2. Run `sudo pvs`, `sudo vgs`, `sudo lvs` — on most cloud images the root filesystem is already LVM-backed by default; confirm whether that's the case for you.

**Success criteria:** You can point to the exact device name of your new, empty disk (e.g. `/dev/sdb`) and distinguish it from your root disk in `lsblk` output.

---

### Lab 2 — Build an LVM stack from scratch

1. Initialize the new disk as a physical volume:
   ```bash
   sudo pvcreate /dev/sdb
   sudo pvs
   ```
2. Create a volume group from it:
   ```bash
   sudo vgcreate vg_lab /dev/sdb
   sudo vgs
   ```
3. Carve out a logical volume smaller than the full VG (leave some free space for Lab 4 — extend within the VG first before attaching a second disk):
   ```bash
   sudo lvcreate -L 2G -n lv_data vg_lab
   sudo lvs
   ```
4. Format and mount it:
   ```bash
   sudo mkfs.ext4 /dev/vg_lab/lv_data
   sudo mkdir -p /mnt/labdata
   sudo mount /dev/vg_lab/lv_data /mnt/labdata
   df -h /mnt/labdata
   ```
5. Make it persistent: find its UUID and add an fstab entry with `nofail` so a mistake here can't block your next boot.
   ```bash
   sudo blkid /dev/vg_lab/lv_data
   echo "UUID=<paste-uuid-here>  /mnt/labdata  ext4  defaults,nofail  0 2" | sudo tee -a /etc/fstab
   sudo umount /mnt/labdata
   sudo mount -a          # verify fstab is valid BEFORE rebooting
   df -h /mnt/labdata
   ```

**Success criteria:** `df -h /mnt/labdata` shows a mounted ~2GB ext4 filesystem, `mount -a` runs cleanly with no errors, and you can explain what each of the six fstab fields you just wrote means.

---

### Lab 3 — Simulate a disk-full scenario

1. Fill the volume close to capacity using `fallocate` (fast, allocates real blocks):
   ```bash
   sudo fallocate -l 1.8G /mnt/labdata/bigfile
   df -h /mnt/labdata
   ```
2. Push it over the edge and observe the failure:
   ```bash
   sudo dd if=/dev/zero of=/mnt/labdata/overflow bs=1M count=500
   # expect: "No space left on device" partway through, and a truncated file
   ```
3. Confirm this is a *space* problem, not an *inode* problem:
   ```bash
   df -h /mnt/labdata      # 100% used, 0 (or near-0) available
   df -i /mnt/labdata      # inode usage should look normal — this rules out inode exhaustion
   ```
4. Now simulate the *other* kind of "disk full" — inode exhaustion — on a small, separate LV so you can tell the two apart. If you have spare VG space:
   ```bash
   sudo lvcreate -L 100M -n lv_inodetest vg_lab
   sudo mkfs.ext4 -N 100 /dev/vg_lab/lv_inodetest   # force only 100 inodes
   sudo mkdir -p /mnt/inodetest
   sudo mount /dev/vg_lab/lv_inodetest /mnt/inodetest
   for i in $(seq 1 100); do sudo touch /mnt/inodetest/file$i; done
   df -h /mnt/inodetest     # plenty of space free
   df -i /mnt/inodetest     # 100% inode usage
   touch /mnt/inodetest/onemore   # fails with "No space left on device" despite free space
   ```
5. Recover: delete the files you created to reclaim space/inodes.
   ```bash
   sudo rm -f /mnt/labdata/bigfile /mnt/labdata/overflow
   df -h /mnt/labdata
   ```

**Success criteria:** You can produce "No space left on device" from *both* a full filesystem and an inode-exhausted filesystem, and explain — using `df -h` vs `df -i` — which one you're looking at without guessing.

---

### Lab 4 — Extend the LVM volume online, without downtime

This is the core hands-on activity for today — do it against the volume you're actively using (`/mnt/labdata`) while it stays mounted the whole time.

1. First, extend using free space still inside the existing volume group (if you left headroom in Lab 2):
   ```bash
   sudo vgs                                        # confirm free space in vg_lab
   sudo lvextend -L +1G /dev/vg_lab/lv_data          # grow the LV by 1G
   sudo resize2fs /dev/vg_lab/lv_data                # grow the ext4 filesystem to match — do NOT skip this
   df -h /mnt/labdata                                # confirm the new size is visible, mount never dropped
   ```
2. Once the VG itself is out of free space, extend the *volume group* by attaching and adding a second disk — this is the realistic "we're out of room, add another disk" scenario:
   ```bash
   sudo pvcreate /dev/sdc                            # your second attached disk
   sudo vgextend vg_lab /dev/sdc
   sudo vgs                                          # confirm more free space is now in the pool
   sudo lvextend -l +100%FREE /dev/vg_lab/lv_data    # use all newly available space
   sudo resize2fs /dev/vg_lab/lv_data
   df -h /mnt/labdata
   ```
3. Throughout both steps, in a second terminal, run a background write to prove the mount never went down:
   ```bash
   while true; do date >> /mnt/labdata/heartbeat.log; sleep 1; done
   ```
   Confirm `heartbeat.log` has no gaps in its timestamps across the entire extend operation.

**Success criteria:** `df -h /mnt/labdata` reports the increased size after each step, the filesystem was never unmounted, and `heartbeat.log` shows continuous, gap-free timestamps proving zero downtime.

---

### Lab 5 — Take and restore from an LVM snapshot

1. Create a snapshot of the live volume:
   ```bash
   sudo lvcreate -L 500M -s -n lv_data_snap /dev/vg_lab/lv_data
   sudo lvs
   ```
2. Mount the snapshot read-only elsewhere and confirm it has the data as of snapshot time:
   ```bash
   sudo mkdir -p /mnt/snap
   sudo mount -o ro /dev/vg_lab/lv_data_snap /mnt/snap
   ls /mnt/snap
   ```
3. Modify the *live* volume (e.g., delete `heartbeat.log`) and confirm the snapshot still shows the old state, proving it's a frozen point-in-time view.
4. Clean up the snapshot:
   ```bash
   sudo umount /mnt/snap
   sudo lvremove /dev/vg_lab/lv_data_snap
   ```

**Success criteria:** You can explain, using what you just observed, why a snapshot lets you back up a live, changing volume consistently — and why an undersized snapshot (too little space reserved for -L) would risk becoming invalid under heavy write load.

---

### Lab 6 — Mount an NFS share and reproduce a stale file handle

*(Needs a second VM, or an NFS server running on the same box for a simplified single-host version — either works for this exercise.)*

1. On the "server" (install and configure): `sudo apt install nfs-kernel-server`, add to `/etc/exports`:
   ```
   /srv/nfsshare  127.0.0.1(rw,sync,no_subtree_check)
   ```
   ```bash
   sudo mkdir -p /srv/nfsshare && sudo chmod 777 /srv/nfsshare
   sudo exportfs -ra
   showmount -e localhost
   ```
2. On the "client" (same box or a second VM), mount it:
   ```bash
   sudo mkdir -p /mnt/nfstest
   sudo mount -t nfs4 localhost:/srv/nfsshare /mnt/nfstest
   echo "hello from client" | sudo tee /mnt/nfstest/test.txt
   ```
3. Reproduce a stale file handle: while a shell is `cd`'d into `/mnt/nfstest`, on the server side remove and recreate the export directory, then re-export:
   ```bash
   # server side
   sudo rm -rf /srv/nfsshare && sudo mkdir -p /srv/nfsshare && sudo chmod 777 /srv/nfsshare
   sudo exportfs -ra
   ```
   ```bash
   # client side, in the shell still cd'd into /mnt/nfstest
   ls          # expect: "Stale file handle"
   ```
4. Recover the client:
   ```bash
   cd /
   sudo umount -f /mnt/nfstest
   sudo mount -t nfs4 localhost:/srv/nfsshare /mnt/nfstest
   ls /mnt/nfstest
   ```

**Success criteria:** You've triggered a genuine "Stale file handle" error and fixed it by unmounting/remounting rather than retrying the same command, and you can explain in one sentence why the client can't recover on its own.

---

### Lab 7 — Watch real disk I/O with `iostat` and `iotop`

1. In one terminal, start a continuous sampler:
   ```bash
   iostat -xz 1
   ```
2. In another terminal, generate real write load against your lab volume:
   ```bash
   sudo dd if=/dev/zero of=/mnt/labdata/iotest bs=1M count=1024 oflag=direct
   ```
3. While that's running, watch `%util` and `await` climb in the `iostat` output for the device backing `vg_lab` (likely `dm-X` for the LVM device-mapper node). Note the read/write throughput columns too.
4. In a third terminal, run `sudo iotop -o` and confirm it identifies `dd` as the process responsible for the I/O you're seeing in `iostat`.
5. Clean up the test file: `sudo rm -f /mnt/labdata/iotest`.

**Success criteria:** You can point to the specific `iostat` columns (`%util`, `await`) that told you the device was under load, and independently confirm via `iotop` which process caused it.

---

### Cleanup

```bash
sudo umount /mnt/labdata /mnt/inodetest 2>/dev/null
sudo umount -f /mnt/nfstest 2>/dev/null
sudo sed -i '/vg_lab\/lv_data\|vg_lab-lv_data/d' /etc/fstab   # remove the lab fstab entry
sudo lvremove -f /dev/vg_lab/lv_inodetest 2>/dev/null
sudo lvremove -f /dev/vg_lab/lv_data
sudo vgremove vg_lab
sudo pvremove /dev/sdb /dev/sdc
# then detach/delete the extra virtual disk(s) from VirtualBox settings or `aws ec2 detach-volume` / `delete-volume`
```

### Stretch challenge

Convert `lv_data` to **thin-provisioned** storage: create a thin pool (`lvcreate -L 4G -T vg_lab/thinpool`) and a thin logical volume from it that's *larger* than the pool itself (e.g. `lvcreate -V 10G -T vg_lab/thinpool -n lv_thin`). Format and mount it, then fill it until you approach the pool's real capacity and observe what happens (`lvs -a` showing data-percent climbing) — explain in your own words why thin provisioning is powerful but dangerous if you don't monitor pool usage closely (a thin pool that fills up can suspend writes across every thin LV built on it, not just the one that triggered it).
