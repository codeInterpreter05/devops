# Day 1 — Cheatsheet: Linux Fundamentals I

## Filesystem hierarchy quick reference

```
/            root of everything
/bin /sbin   essential binaries (often -> /usr/bin, /usr/sbin)
/usr         installed software (bin, lib, share)
/usr/local   manually installed software (outside package manager)
/etc         config files (no binaries)
/var         variable/runtime data
/var/log     logs
/var/lib     persistent app state (e.g. /var/lib/docker)
/tmp         ephemeral scratch space (may be wiped on reboot)
/home        user home directories
/root        root user's home (not under /home)
/opt         third-party/vendor software bundles
/proc        virtual — live kernel & process info
/sys         virtual — live kernel/device state
/dev         device files
/mnt /media  manual / removable-media mount points
/boot        kernel, initramfs, bootloader
```

## Path basics

```bash
pwd                # print current working directory
cd /absolute/path   # go anywhere, unambiguous
cd relative/path    # relative to pwd
cd ..               # up one level
cd ../..            # up two levels
cd -                # toggle to previous directory
cd ~ | cd           # go home
```

## `ls`

```bash
ls -la          # long + hidden files
ls -lh          # human-readable sizes
ls -lart        # long, all, reverse-time sort (newest last)
ls -d */        # directories only
```

## `cp` / `mv` / `rm`

```bash
cp file dest             # copy file
cp -r dir/ dest/          # copy directory (recursive required)
cp -a dir/ dest/          # archive copy — preserves perms/timestamps/symlinks
cp -iv src dest            # interactive + verbose

mv old new                # rename or move
mv -i old new               # prompt before overwrite

rm file                    # delete file — NO UNDO
rm -r dir/                  # recursive delete
rm -rf dir/                 # recursive + force — dangerous, no prompts
rm -i file                  # prompt before delete
rm -rf -- "$DIR"            # safe pattern: quote + -- to stop flag injection
```

## `find`

```bash
find /etc -name "*.conf"              # by name (case-sensitive)
find /etc -iname "*.conf"             # case-insensitive
find / -maxdepth 2 -type d             # limit depth, dirs only
find / -type f                         # files only
find /var/log -mtime -1               # modified within last 1 day
find /tmp -mtime +7                    # older than 7 days
find / -size +100M                     # larger than 100MB
find /tmp -mtime +7 -delete            # delete matches (careful!)
find . -name "*.log" -exec grep -l ERROR {} \;    # act on each match
find . -name "*.log" -print0 | xargs -0 grep -l ERROR   # same, via xargs
```

## Getting help

```bash
man <command>          # full manual (section 1 by default)
man 5 <name>           # file-format section (e.g. man 5 crontab)
man -k <keyword>       # search descriptions of all man pages (= apropos)
tldr <command>         # fast, example-driven cheat sheet
tldr --update          # refresh tldr's local cache
```
