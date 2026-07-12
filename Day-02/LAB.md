# Day 2 — Lab: Linux Fundamentals II

**Goal:** Build a real multi-user scenario from scratch — create users and groups, restrict file access with `chmod`/`chown`, verify access boundaries by actually switching identities with `su`, and observe SUID/SGID/sticky bit behavior firsthand.

**Prerequisites:** Root (or sudo) access on a **disposable** Linux environment — this lab creates real system users/groups and modifies real files, so use a throwaway VM or container, never your primary machine. A quick disposable option:
```bash
docker run -it --rm ubuntu:22.04 bash
apt update && apt install -y sudo passwd util-linux vim
```
Everything below assumes you're starting as root inside that environment.

---

### Lab 1 — Create the multi-user scenario

1. Create two regular users with home directories and a login shell:
   ```bash
   useradd -m -s /bin/bash alice
   useradd -m -s /bin/bash bob
   useradd -m -s /bin/bash carol
   ```
2. Set passwords non-interactively (fine for a disposable lab; never do this on a real system):
   ```bash
   echo "alice:Passw0rd!" | chpasswd
   echo "bob:Passw0rd!"   | chpasswd
   echo "carol:Passw0rd!" | chpasswd
   ```
3. Confirm each account exists with the expected identity:
   ```bash
   id alice
   id bob
   id carol
   tail -3 /etc/passwd
   ls -ld /home/alice /home/bob /home/carol
   ```
4. Lock down each home directory so only its owner can enter it (don't rely on distro defaults):
   ```bash
   chmod 700 /home/alice /home/bob /home/carol
   ```

**Success criteria:** `id alice`, `id bob`, and `id carol` each return a valid UID/GID with no errors, all three home directories exist and are owned by their respective users, and `ls -ld` shows `drwx------` for each.

---

### Lab 2 — Set up a shared group

1. Create a group for a "project team" containing alice and bob, but not carol:
   ```bash
   groupadd devteam
   usermod -aG devteam alice
   usermod -aG devteam bob
   ```
2. Verify membership (and confirm carol is correctly excluded):
   ```bash
   id -nG alice
   id -nG bob
   id -nG carol
   ```
3. Create a shared directory owned by root, group `devteam`, with the SGID bit set so anything created inside inherits the `devteam` group automatically:
   ```bash
   mkdir /srv/devteam
   chown root:devteam /srv/devteam
   chmod 2770 /srv/devteam
   ls -ld /srv/devteam
   ```

**Success criteria:** `id -nG alice` and `id -nG bob` both list `devteam`; `id -nG carol` does not. `ls -ld /srv/devteam` shows `drwxrws---` with group `devteam`.

---

### Lab 3 — Core hands-on: restrict access and verify with `su`

This is the assigned hands-on activity for today — do it for real, switching identities with `su` rather than just reasoning about permissions on paper.

1. As alice, create a private file only she can read:
   ```bash
   su - alice -c 'echo "alice private notes" > ~/private.txt && chmod 600 ~/private.txt'
   ```
2. Confirm bob is denied access to it (blocked by alice's `700` home directory, independent of the file's own permissions):
   ```bash
   su - bob -c 'cat /home/alice/private.txt'
   # expect: Permission denied
   ```
3. Now use the shared `devteam` directory instead. As alice, create a file there:
   ```bash
   su - alice -c 'echo "sprint plan v1" > /srv/devteam/plan.txt'
   ls -l /srv/devteam/plan.txt
   ```
   Note the file's **group** is `devteam` even though alice's primary group is `alice` — that's the SGID bit on the parent directory in action.
4. As bob (a `devteam` member), append to the same file — should succeed because of group `rw`:
   ```bash
   su - bob -c 'echo "bob addition" >> /srv/devteam/plan.txt'
   cat /srv/devteam/plan.txt
   ```
5. As carol (not a `devteam` member), attempt the same thing — should fail:
   ```bash
   su - carol -c 'cat /srv/devteam/plan.txt'
   su - carol -c 'echo "carol trying" >> /srv/devteam/plan.txt'
   # expect: Permission denied on both
   ```
6. As root, tighten `plan.txt` itself to read-only for the group and confirm bob can no longer append:
   ```bash
   chmod 640 /srv/devteam/plan.txt
   su - bob -c 'echo "should now fail" >> /srv/devteam/plan.txt'
   # expect: Permission denied
   su - bob -c 'cat /srv/devteam/plan.txt'
   # expect: succeeds (read still allowed)
   ```

**Success criteria:** bob is blocked from alice's home directory entirely; bob can read/write the shared file while it's group-writable and loses write access the moment it's `chmod 640`; carol is blocked from the shared directory at every step; the file created by bob shows group `devteam`, proving SGID inheritance.

---

### Lab 4 — SUID and the sticky bit, hands-on

1. Inspect a real SUID binary and use it as intended:
   ```bash
   ls -l /usr/bin/passwd
   # -rwsr-xr-x ... /usr/bin/passwd
   su - alice -c 'passwd'   # alice changes HER OWN password, writing to root-only /etc/shadow
   ```
2. Build a minimal SUID binary yourself to observe effective-UID elevation directly (educational only — clean this up immediately after, see Cleanup):
   ```bash
   cp /bin/cat /usr/local/bin/suid_cat
   chown root:root /usr/local/bin/suid_cat
   chmod 4755 /usr/local/bin/suid_cat
   ls -l /usr/local/bin/suid_cat
   su - alice -c '/usr/local/bin/suid_cat /etc/shadow | head -2'
   ```
   Alice can read a root-only file through the SUID copy of `cat`, even though `cat /etc/shadow` directly (without SUID) would fail for her. This is exactly why unaudited SUID binaries are a real security risk — immediately remove it once you've observed the behavior.
3. Confirm the kernel ignores SUID on scripts (not just binaries):
   ```bash
   printf '#!/bin/bash\ncat /etc/shadow\n' > /usr/local/bin/suid_script.sh
   chmod 4755 /usr/local/bin/suid_script.sh
   su - alice -c '/usr/local/bin/suid_script.sh'
   # expect: Permission denied — SUID has no effect on shebang scripts
   ```
4. Observe the sticky bit's deletion protection on `/tmp`:
   ```bash
   ls -ld /tmp
   su - alice -c 'echo "alice tmp file" > /tmp/alice_file.txt'
   su - bob -c 'rm /tmp/alice_file.txt'
   # expect: Permission denied (bob can't delete alice's file despite /tmp being 1777)
   su - alice -c 'rm /tmp/alice_file.txt'
   # expect: succeeds — the owner can always remove their own file
   ```
5. Audit the system for existing SUID binaries (standard security hygiene):
   ```bash
   find / -perm -4000 -type f 2>/dev/null
   ```

**Success criteria:** the SUID `cat` copy successfully reads `/etc/shadow` as alice; the SUID script does **not** elevate privilege; bob cannot delete alice's file in `/tmp` but alice can; the `find` audit command runs and lists SUID binaries on the box.

---

### Lab 5 — sudo and sudoers

1. Grant alice passwordless permission to restart one specific service, using a dedicated file in `sudoers.d` (never edit `/etc/sudoers` directly):
   ```bash
   visudo -f /etc/sudoers.d/alice-nginx
   ```
   Add this line, save, and exit:
   ```
   alice ALL=(root) NOPASSWD: /usr/bin/systemctl restart nginx
   ```
2. Confirm alice can run exactly that command without a password, and nothing broader:
   ```bash
   su - alice -c 'sudo -l'
   su - alice -c 'sudo systemctl restart nginx'
   su - alice -c 'sudo whoami'
   # expect: the last command prompts for/denies a password — alice is NOT granted general sudo
   ```
3. Compare `su` vs `sudo` directly: as alice, try `su - bob` and observe it asks for **bob's** password (which alice doesn't know), while `sudo -u bob whoami` (if you grant it) would instead ask for **alice's own** password.

**Success criteria:** `sudo -l` as alice shows only the one scoped rule; the scoped command runs without a password prompt; `sudo whoami` (unscoped) is denied or prompts for a password alice can't satisfy for that command.

---

### Cleanup

Remove everything created in this lab so the environment (or your notes on a persistent VM) doesn't accumulate test artifacts:

```bash
rm -f /usr/local/bin/suid_cat /usr/local/bin/suid_script.sh
rm -f /etc/sudoers.d/alice-nginx
rm -rf /srv/devteam
userdel -r alice
userdel -r bob
userdel -r carol
groupdel devteam
```

(If using the throwaway Docker container from the prerequisites, simply exiting the shell and letting `--rm` destroy the container accomplishes the same thing.)

### Stretch challenge

Design a sudoers rule that lets a `deploy` user run **only** `/usr/bin/systemctl restart myapp` and `/usr/bin/systemctl status myapp` as root, without a password, but explicitly ensure that user still cannot read `/etc/shadow`, cannot run arbitrary commands via `sudo -u`, and cannot use `sudo` to spawn an interactive root shell. Verify each of those three restrictions by attempting them as the `deploy` user and confirming denial.
