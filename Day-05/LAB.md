# Day 5 — Lab: Bash Scripting I

**Goal:** Write, harden, and test a real backup script — one that tars up a directory, rotates old backups (keeping only the last 5), and logs every step with timestamps — using `set -euo pipefail`, and validate it with `shellcheck`.

**Prerequisites:** A Linux/macOS shell with bash 4+ (`bash --version`). Install `shellcheck`: `sudo apt install shellcheck` (Debian/Ubuntu), `brew install shellcheck` (macOS), or `sudo dnf install ShellCheck` (Fedora/RHEL). Have a text editor with bash syntax support — `vim` is the assigned editor for this phase (`sudo apt install vim` / `brew install vim`).

---

### Lab 1 — Skeleton: shebang + `set -euo pipefail`

1. Create a working directory and the script file:
   ```bash
   mkdir -p ~/bash-lab && cd ~/bash-lab
   vim backup.sh
   ```
2. Write the skeleton — just the shebang, the safety flags, and a placeholder:
   ```bash
   #!/usr/bin/env bash
   set -euo pipefail

   echo "backup.sh skeleton is alive"
   ```
3. Make it executable and run it directly (exercises the shebang/kernel exec path, not just `bash backup.sh`):
   ```bash
   chmod +x backup.sh
   ./backup.sh
   ```
4. Prove `set -u` works: temporarily add `echo "$UNDEFINED_VAR"` to the script, run it, and observe it exits with an error instead of printing a blank line. Remove that line afterward.

**Success criteria:** `./backup.sh` runs via the shebang (not `bash backup.sh`) and prints the placeholder message. You've confirmed `set -u` catches an unset variable reference.

---

### Lab 2 — Backup logic with `tar`

1. Create a fake source directory with some content to back up:
   ```bash
   mkdir -p ~/bash-lab/data/project
   echo "hello world" > ~/bash-lab/data/project/file1.txt
   echo "more content" > ~/bash-lab/data/project/file2.txt
   mkdir -p ~/bash-lab/backups
   ```
2. Extend `backup.sh` to accept a source dir and backup dir as arguments, validate the source exists, and create a timestamped `tar.gz`:
   ```bash
   #!/usr/bin/env bash
   set -euo pipefail

   SOURCE_DIR="${1:?Usage: $0 <source_dir> <backup_dir>}"
   BACKUP_DIR="${2:?Usage: $0 <source_dir> <backup_dir>}"
   TIMESTAMP="$(date +%Y%m%d-%H%M%S)"
   BACKUP_NAME="backup-${TIMESTAMP}.tar.gz"

   if [[ ! -d "$SOURCE_DIR" ]]; then
     echo "ERROR: source directory '$SOURCE_DIR' does not exist" >&2
     exit 1
   fi

   mkdir -p "$BACKUP_DIR"

   tar -czf "${BACKUP_DIR}/${BACKUP_NAME}" -C "$(dirname "$SOURCE_DIR")" "$(basename "$SOURCE_DIR")"
   echo "Created ${BACKUP_DIR}/${BACKUP_NAME}"
   ```
3. Run it and confirm a `.tar.gz` file was created:
   ```bash
   ./backup.sh ~/bash-lab/data/project ~/bash-lab/backups
   ls -lh ~/bash-lab/backups
   tar -tzf ~/bash-lab/backups/backup-*.tar.gz    # list contents without extracting
   ```
4. Run it again with a nonexistent source directory (`./backup.sh /no/such/dir ~/bash-lab/backups`) and confirm it fails with your custom error message and a non-zero exit code (`echo $?`).

**Success criteria:** The script produces a valid, timestamped tarball of the source directory, using `-C`/basename so the archive doesn't embed absolute host paths, and fails cleanly with a clear message when the source doesn't exist.

---

### Lab 3 — Rotation: keep only the last 5 backups

1. Generate backup "history" quickly by running the script several times in a row (sleep briefly so timestamps differ):
   ```bash
   for i in $(seq 1 8); do ./backup.sh ~/bash-lab/data/project ~/bash-lab/backups; sleep 1; done
   ls ~/bash-lab/backups | wc -l
   ```
2. Add rotation logic to `backup.sh`, right after the `tar` step, that deletes everything except the 5 newest backups:
   ```bash
   MAX_BACKUPS=5

   mapfile -t backups < <(find "$BACKUP_DIR" -maxdepth 1 -name 'backup-*.tar.gz' -printf '%T@ %p\n' | sort -rn | awk '{print $2}')

   if (( ${#backups[@]} > MAX_BACKUPS )); then
     for old in "${backups[@]:MAX_BACKUPS}"; do
       echo "Removing old backup: $old"
       rm -f -- "$old"
     done
   fi
   ```
3. Re-run the script a few more times and confirm the count never exceeds 5:
   ```bash
   ./backup.sh ~/bash-lab/data/project ~/bash-lab/backups
   ls ~/bash-lab/backups | wc -l   # should stay at 5
   ```

**Success criteria:** No matter how many times the script runs, exactly the 5 most recent `backup-*.tar.gz` files remain — older ones are deleted automatically.

---

### Lab 4 — Timestamped logging

1. Add a `log()` helper near the top of the script (after the `set` line) that prefixes every message with a timestamp and level, and writes to both the terminal and a log file:
   ```bash
   LOG_FILE="${BACKUP_DIR}/backup.log"

   log() {
     local level="$1"; shift
     printf '%s [%s] %s\n' "$(date '+%Y-%m-%d %H:%M:%S')" "$level" "$*" | tee -a "$LOG_FILE"
   }
   ```
   Note: `LOG_FILE` depends on `BACKUP_DIR`, which is set from `$2` — make sure the `log` function definition comes *after* `BACKUP_DIR` is assigned, and that `mkdir -p "$BACKUP_DIR"` runs before the first `log` call.
2. Replace every plain `echo` in the script with a call to `log INFO "..."` or `log ERROR "..."` (for the error branch), for example:
   ```bash
   log INFO "Starting backup of '$SOURCE_DIR' -> '${BACKUP_DIR}/${BACKUP_NAME}'"
   ...
   log INFO "Removing old backup: $old"
   ...
   log INFO "Backup complete. $(find "$BACKUP_DIR" -maxdepth 1 -name 'backup-*.tar.gz' | wc -l) backups retained."
   ```
3. Run the script twice and inspect the log:
   ```bash
   ./backup.sh ~/bash-lab/data/project ~/bash-lab/backups
   cat ~/bash-lab/backups/backup.log
   ```

**Success criteria:** Every run appends timestamped, leveled lines to `backup.log`, and the same messages appear on screen (via `tee -a`) so the script is usable both interactively and as a background/cron job.

---

### Lab 5 — Run `shellcheck` and fix every warning

1. Run shellcheck against the finished script:
   ```bash
   shellcheck backup.sh
   ```
2. For each finding, look up its `SC####` code (shellcheck prints a link) and fix it rather than suppressing it — common ones you're likely to see at this stage: unquoted expansions (`SC2086`), using `local` before declaring `set -u`-safe defaults, or a `useless cat`. Only add an inline `# shellcheck disable=SCxxxx` comment if you've deliberately decided the warning doesn't apply, and note *why* in a comment.
3. Re-run `shellcheck backup.sh` until it reports no warnings.

**Success criteria:** `shellcheck backup.sh` exits 0 with no output.

---

### Lab 6 — Test the failure paths on purpose

Don't just test the happy path — prove the script fails safely.

1. **Missing arguments**: run `./backup.sh` with no arguments. Confirm you get the `Usage:` message from `${1:?...}` and a non-zero exit code, not a confusing `unbound variable` stack of errors.
2. **Nonexistent source**: run `./backup.sh /path/does/not/exist ~/bash-lab/backups`. Confirm the explicit check catches it before `tar` ever runs.
3. **Unwritable backup destination**: create a directory you don't have write permission to (`mkdir -p /tmp/no-write && chmod 000 /tmp/no-write`), then run `./backup.sh ~/bash-lab/data/project /tmp/no-write` and confirm the script exits non-zero instead of continuing silently. Restore permissions afterward (`chmod 755 /tmp/no-write`) so cleanup can remove it.
4. **Interrupt mid-run**: add a `trap 'log ERROR "Interrupted (signal), exiting"; exit 130' INT TERM` line near the top, start a backup on a large enough directory to take a moment, and press Ctrl-C. Confirm the trap message logs before the process exits, and check `echo $?` shows `130`.
5. **Confirm `pipefail` matters**: temporarily pipe an intentionally-failing command into a successful one without `pipefail` active (comment out `set -o pipefail` and test `false | true; echo $?`), see it print `0`, then restore `set -euo pipefail` and confirm the same pipeline now reports failure.

**Success criteria:** Every failure mode above produces a clear log message and a non-zero exit code — nothing fails silently, and nothing continues past a check that should have stopped it.

---

### Cleanup

```bash
rm -rf ~/bash-lab
```

### Stretch challenge

Parameterize `MAX_BACKUPS` as an optional third argument (default 5 if not supplied, using `${3:-5}`), add a `--dry-run` flag that logs what rotation *would* delete without actually deleting anything, and wire the whole script into a `cron` entry that runs it nightly at 2 a.m. against a real directory on your machine — verify with `grep CRON /var/log/syslog` (or `journalctl -u cron`) the next morning that it actually fired.
