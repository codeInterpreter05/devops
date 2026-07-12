# Day 1 — Quiz: Linux Fundamentals I

Try to answer without looking at your notes. Answers are at the bottom.

1. What is the Filesystem Hierarchy Standard (FHS), and is it enforced by the kernel?
2. Where do system-wide configuration files live? Where do logs live? Where does persistent application state (like a database's data directory) typically live?
3. What's the difference between `/tmp` and `/var/lib` in terms of what you should trust them to keep?
4. Why does `cat /proc/cpuinfo` return content instantly even though `ls -la` shows the file as 0 bytes?
5. What's the difference between an absolute path and a relative path? Give an example of each.
6. Why is a script that runs `rm -rf ./build` risky when triggered from cron or systemd?
7. What does `cp -a` preserve that plain `cp -r` does not guarantee?
8. What's the difference between `find` and `locate`? When would you choose one over the other?
9. Write a `find` command that lists all `.log` files under `/var/log` modified in the last 2 days.
10. What's the difference between `man <command>` and `man 5 <command>`? Give a concrete example where this distinction matters.
11. Why might `tldr` be dangerous to rely on exclusively for a command like `rm` or `dd`?
12. **Interview question:** Explain the Linux filesystem hierarchy and where configs/logs live, as if to an interviewer who wants to know you've actually operated Linux systems, not just read about them.

---

## Answers

1. FHS is a convention followed by distro maintainers describing what each top-level directory (`/etc`, `/var`, `/usr`, etc.) is for. It is **not enforced by the kernel** — nothing stops you from putting a config file in `/tmp`, but tooling, packages, and other engineers all assume FHS conventions hold.
2. Configs: `/etc`. Logs: `/var/log`. Persistent app state: `/var/lib` (e.g., `/var/lib/docker`, `/var/lib/mysql`).
3. `/tmp` is ephemeral scratch space that may be cleared on reboot or by a scheduled cleanup (`systemd-tmpfiles`) — never store anything there you can't regenerate. `/var/lib` is meant to persist across reboots and hold real application state that should be backed up.
4. `/proc` is a **virtual filesystem** — its "files" are generated on the fly by the kernel when read, not stored on disk. The 0-byte size is because the kernel doesn't pre-compute a size; content is produced dynamically at read time.
5. Absolute path starts at `/` and is unambiguous regardless of current directory, e.g. `/etc/nginx/nginx.conf`. Relative path is interpreted against your current working directory, e.g. `nginx.conf` or `../nginx/nginx.conf` — same string, different target depending on where you are.
6. Cron/systemd jobs often start in an unexpected working directory (commonly `/` or `/root`), not the directory you had in mind when writing the script. A relative path like `./build` could resolve somewhere completely different, and combined with `-rf`, that's destructive with no undo.
7. `cp -a` (archive mode) preserves permissions, ownership, timestamps, and symlinks as-is. Plain `cp -r` copies the file contents and structure but can change ownership/permissions to defaults and dereference symlinks.
8. `find` walks the actual filesystem live at the moment you run it — always accurate, but potentially slow on huge trees. `locate` queries a pre-built index (`updatedb`) — very fast, but can return stale results if files changed since the last index build. Use `find` for precise/current/condition-based searches (by time, size, permission); use `locate` for quick "does this exist somewhere" lookups.
9. `find /var/log -name "*.log" -mtime -2`
10. `man <command>` defaults to section 1 (user commands). `man 5 <command>` explicitly requests section 5 (file formats). Example: `man crontab` shows the `crontab` command's usage; `man 5 crontab` shows the *file format* of a crontab file (fields, syntax) — very different, useful content under the same name.
11. `tldr` intentionally shows only the most common usage patterns and omits many flags — for destructive commands (`rm`, `dd`, `mkfs`), missing an important flag's behavior (like what `dd`'s `of=` does to an entire disk) can cause real damage. Verify unfamiliar or destructive commands in `man` first.
12. Strong answer: "Linux uses a single tree rooted at `/`, with everything else mounted into it. Configuration lives in `/etc`, logs in `/var/log`, persistent application/service state in `/var/lib`, ephemeral scratch data in `/tmp`, user data in `/home`, and third-party/vendor software often in `/opt`. `/proc` and `/sys` are virtual filesystems that expose live kernel and process state rather than data stored on disk — that's why, for example, `/proc/<pid>/status` always reflects a process's current state in real time." Follow up by mentioning you've used this to debug things (e.g., finding a runaway process's open file descriptors via `/proc/<pid>/fd`) if you have a real example.
