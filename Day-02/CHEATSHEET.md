# Day 2 — Cheatsheet: Linux Fundamentals II

## Identity files

```
/etc/passwd    username:x:UID:GID:GECOS:home:shell   (world-readable, no real password)
/etc/shadow    username:hash:lastchg:min:max:warn:inactive:expire   (root-only)
/etc/group     groupname:x:GID:member1,member2       (SECONDARY members only)
/etc/gshadow   group passwords/admins (rarely used)
/etc/login.defs   UID_MIN/UID_MAX and other account creation defaults
/etc/nsswitch.conf   where identity lookups resolve from (files, LDAP, SSSD, etc.)
```

## Identity commands

```bash
whoami            # current EFFECTIVE username
who am i           # ORIGINAL login session (from utmp) — diverges from whoami after su
logname            # original authenticated login name
id                 # uid, gid, all groups
id -u / id -g       # numeric UID / primary GID only
id -nG              # names of ALL groups (primary + secondary)
id alice            # identity of a specific user
groups [user]        # shorthand group membership
getent passwd alice   # NSS-aware lookup (works with LDAP/SSSD too)
```

## User & group management

```bash
useradd -m -s /bin/bash alice     # create user, home dir, shell
passwd alice                       # set/change password
chpasswd <<< "alice:Passw0rd!"      # non-interactive password set (labs/automation only)
usermod -aG group user              # ADD to secondary group (-a REQUIRED or you wipe other groups)
usermod -s /usr/sbin/nologin user    # disable interactive login (service accounts)
userdel -r alice                      # delete user + home dir + mail spool
groupadd devteam                       # create group
groupdel devteam                        # delete group (fails if it's someone's primary group)
```

## Permission model (rwx)

```
r = 4   w = 2   x = 1
File:      r=read contents        w=modify/truncate        x=execute as program
Directory: r=list filenames        w=create/delete/rename entries inside   x=traverse (cd into it)
```
Check order: **owner match → owner bits (exclusive)** ; else **group match → group bits** ; else **other bits**. No fallback between categories.

## `chmod`

```bash
chmod 755 file       # rwxr-xr-x
chmod 644 file        # rw-r--r--
chmod 700 dir           # rwx------
chmod 600 id_rsa         # rw------- (secrets/keys)
chmod u+x file             # symbolic: add owner execute
chmod g-w file              # symbolic: remove group write
chmod o=r file                # symbolic: set other to read only
chmod -R g+rX dir/             # recursive; X = execute only where already executable/dir (safe)
```

## `chown` / `chgrp`

```bash
chown alice file              # change owner (root-only for arbitrary target user)
chown alice:devteam file       # change owner + group together
chown :devteam file             # change group only
chgrp devteam file                # equivalent group-only change (must belong to target group if non-root)
chown -R user:group dir/            # recursive — verify symlink behavior before running
```

## `umask`

```bash
umask               # show current mask, e.g. 0022
umask 022            # new dirs -> 755, new files -> 644
umask 077             # new dirs -> 700, new files -> 600 (private-by-default hardening)
```

## Special bits

```bash
chmod 4755 file        # SUID  (s in owner-execute slot)
chmod 2775 dir           # SGID  (s in group-execute slot)
chmod 1777 dir             # sticky (t in other-execute slot)
chmod u+s file               # symbolic SUID
chmod g+s dir                  # symbolic SGID
chmod +t dir                     # symbolic sticky

find / -perm -4000 -type f 2>/dev/null   # audit: all SUID binaries
find / -perm -2000 -type f 2>/dev/null    # audit: all SGID binaries
find / -perm -1000 -type d 2>/dev/null     # audit: all sticky directories
```
SUID: process runs with **file owner's effective UID** (binaries only — kernel ignores it on shebang scripts).
SGID (dir): new files/subdirs **inherit the directory's group**.
Sticky (dir): only the **file owner / dir owner / root** can delete or rename inside, even if the directory is world-writable.

## `su` vs `sudo`

```bash
su               # switch to root, KEEP your shell's environment — needs ROOT's password
su -              # switch to root as a LOGIN shell, reset environment — needs ROOT's password
su - alice          # switch to alice as login shell — needs ALICE's password

sudo <cmd>            # run ONE command as root — needs YOUR OWN password, logged per-invocation
sudo -u alice <cmd>     # run a command as an arbitrary user
sudo -l                  # list what YOU are allowed to run
sudo -i                    # simulate root login shell (like `su -`, via sudo's audit path)
sudo -s                     # root shell, keep current environment
```

## sudoers

```bash
visudo                      # ALWAYS use this to edit /etc/sudoers (syntax-checked, locked)
visudo -f /etc/sudoers.d/x    # edit a drop-in file instead of the monolithic sudoers file
```
```
# user  host = (runas_user)  commands
alice   ALL = (ALL:ALL) ALL
%sudo   ALL = (ALL) ALL              # Debian/Ubuntu convention group
%wheel  ALL = (ALL) ALL               # RHEL/Fedora convention group
deploy  ALL = NOPASSWD: /usr/bin/systemctl restart myapp   # scope NOPASSWD narrowly, never to ALL
```

## Audit / log locations

```
/var/log/auth.log     # Debian/Ubuntu — sudo & auth events
/var/log/secure         # RHEL/CentOS — sudo & auth events
```
