# Day 18 — Resources: Storage & Filesystems

## Primary (assigned)

- **AWS Storage documentation** (docs.aws.amazon.com — EBS, EFS, and S3 user guides) — the assigned starting point for this day. Read the "How it works" and "Storage classes/types" sections for EBS, EFS, and S3 specifically; that's the material the interview question in this day's quiz is drawn from.

## Deepen your understanding

- **Red Hat's LVM Administration Guide** (access.redhat.com/documentation, "Configuring and managing logical volumes") — the most thorough, vendor-neutral treatment of PV/VG/LV mechanics, thin provisioning, and snapshots; applies to any distro running LVM, not just RHEL.
- **`man lvm`, `man fstab`, `man nfs`** — run these directly on any Linux box. `man fstab` in particular documents every valid mount option, not just the common ones covered in this day's notes.
- **AWS EBS Volume Types documentation** — the exact IOPS/throughput/burst-credit mechanics for `gp2` vs `gp3` vs `io1`/`io2` vs `st1`/`sc1`, worth reading closely since the tradeoffs shift as AWS updates volume types.
- **sysstat project documentation** (github.com/sysstat/sysstat) — the authoritative reference for every `iostat` column, including ones this day's notes didn't cover (e.g., per-CPU stats via `mpstat`, historical data via `sar`).

## Reference / lookup

- `man 5 fstab` — the fstab file format specifically (section 5, not the default section 1).
- `man 8 mount`, `man 8 lvcreate`, `man 8 lvextend`, `man 8 resize2fs`, `man 8 xfs_growfs` — admin-command manuals (section 8) for every command used in this day's lab.
- **explainshell.com** — paste any `lvcreate`/`mount`/`iostat` invocation from this day's cheatsheet to see every flag broken down inline.

## Practice

- **Killercoda / Katacoda-style free Linux scenarios** (killercoda.com) — several free, browser-based scenarios let you practice LVM resizing and disk management on disposable throwaway VMs without needing your own cloud account.
- **A free-tier AWS account** — create and attach a real EBS volume, mount an EFS filesystem from two EC2 instances at once, and compare the mount experience directly; there's no substitute for seeing EBS's single-attach behavior and EFS's multi-AZ behavior side by side on real infrastructure.
