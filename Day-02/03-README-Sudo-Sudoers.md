# Day 2 — Linux Fundamentals II: sudo, su & Privilege Escalation

**Phase:** 0 – Foundation | **Week:** W1 | **Domain:** Linux | **Flag:** ⚡ Interview-critical

## Brief

Root can do anything — bypass every permission check, read every file, kill every process. That's exactly why production systems don't hand out the root password: instead, they control **how** and **with what audit trail** someone temporarily acts as root (or as some other user). `sudo` and `su` are the two mechanisms for this, and the difference between them is one of the most common "do you actually operate servers" interview checks — because in real incident response, "who ran this command and when" often matters as much as "what did it do."

## `su` vs `sudo` — fundamentally different models

- **`su [user]`** ("switch user") starts a **new login session as the target user**, authenticated with the **target user's** password. Default target is root. It effectively hands you a shell that *is* that user until you exit.
- **`sudo <command>`** runs a **single command** as another user (root by default), authenticated with **your own** password (verifying you're in the sudoers policy, not that you know root's password — root may not even have a usable password at all on a well-configured server). Every invocation is logged.

```bash
su               # switch to root, keep YOUR current shell's environment (PATH, cwd, etc.)
su -             # switch to root as a full LOGIN shell — resets environment to root's own (PATH, HOME, etc.)
su - alice       # switch to alice as a login shell
sudo whoami       # run a single command as root, using YOUR password
sudo -u alice ls  # run a single command as an arbitrary user, not just root
sudo -i           # simulate an initial login shell as root (like `su -` but via sudo's auth/audit path)
sudo -s           # spawn a root shell, but keep your CURRENT environment
sudo -l            # list what commands YOU are permitted to run via sudo
```

The `su` vs `su -` distinction is a real, recurring bug source: **plain `su`** keeps your current shell's environment variables (including `PATH`), so a `PATH` pointing at your own `~/bin` can shadow system binaries even while "acting as root." **`su -`** (or `su -l`) resets the environment to match the target user's login environment exactly as if they'd logged in fresh — the safer default when you actually need to *be* that user, not just borrow a shell.

## Why sudo is preferred operationally

1. **No shared secret.** Nobody needs to know the root password (many hardened systems lock it entirely — `passwd -l root` — making `su` to root impossible even if attempted).
2. **Per-command audit trail.** Every `sudo` invocation is logged via syslog, typically to `/var/log/auth.log` (Debian/Ubuntu) or `/var/log/secure` (RHEL/CentOS), recording who ran what, as whom, and when — `su` only logs the *session switch*, not each subsequent command.
3. **Fine-grained authorization.** Sudoers policy can grant a user rights to run *specific* commands as *specific* users, rather than all-or-nothing root access.
4. **Time-boxed elevation.** By default sudo caches a "ticket" (typically 15 minutes) so you're not re-prompted for every command in a burst, but the elevation naturally expires — unlike an `su` session, which stays elevated until you explicitly exit.

## `/etc/sudoers` and `visudo`

The sudoers file defines who can run what, as whom. **Never edit it directly with a plain text editor** — always use `visudo`:

```bash
sudo visudo
```

`visudo` locks the file against concurrent edits, and — critically — **validates syntax before saving**. A malformed `/etc/sudoers` edited directly can break `sudo` for everyone, including you, potentially leaving the box needing single-user-mode/console recovery to fix. `visudo` refuses to save a broken file and reprompts you.

### Sudoers syntax

```
# user   host = (runas_user:runas_group)  commands
alice    ALL = (ALL:ALL) ALL
%devops  ALL = (ALL) ALL
bob      ALL = (root) /usr/bin/systemctl restart nginx, /usr/bin/systemctl status nginx
deploy   ALL = NOPASSWD: /usr/bin/systemctl restart myapp
```

- `%groupname` (leading `%`) grants the rule to everyone in that group — e.g. `%sudo` (Debian/Ubuntu convention) or `%wheel` (RHEL/Fedora convention) is how "add user to sudo group" actually works under the hood; `usermod -aG sudo alice` just adds alice to the group this rule already trusts.
- `(ALL:ALL)` means "as any user, as any group" — restricting this to `(root)` or a specific service account narrows the blast radius of what an authorized user can impersonate.
- `NOPASSWD:` skips the password prompt for matched commands — convenient for automation (CI runners, deploy scripts) but a common **over-privileging** mistake when applied to `ALL` commands instead of a narrow, specific list.
- Command paths should be **absolute** (`/usr/bin/systemctl`, not `systemctl`) — sudoers evaluates the literal command, and an unqualified name is both ambiguous and a path-hijack risk.

### `/etc/sudoers.d/` — the preferred way to add rules

Rather than editing the monolithic `/etc/sudoers`, drop a dedicated file into `/etc/sudoers.d/` (included automatically via `#includedir /etc/sudoers.d` at the bottom of the main file):

```bash
sudo visudo -f /etc/sudoers.d/deploy-user
```

This keeps custom policy isolated, easy to review/diff/version-control per-purpose, and easy to remove cleanly (`rm` one file instead of untangling lines from a shared one). Filenames in `sudoers.d` must not contain a `.` (dot) or a `~` — the parser skips files matching those (they're treated as backups/temp files), a subtle gotcha if you name a file `deploy.conf`.

## Security hygiene around sudo

- `env_reset` is the sane default (strips most environment variables before running the command) — it exists because a malicious or stale `LD_PRELOAD`, `PATH`, or similar variable inherited from your shell could otherwise influence what actually runs *as root*.
- Broad `NOPASSWD: ALL` rules are a textbook privilege-escalation vector: if an attacker compromises that user's session, they get instant, unaudited root. Scope `NOPASSWD` to the exact command(s) automation needs, never to `ALL`.
- `sudo -l` is a first move during a security review or incident: it tells you exactly what the *current* account is authorized to do without reading sudoers directly.
- sudo integrates with **PAM** (Pluggable Authentication Modules) for the actual authentication step, so PAM policy (password complexity, MFA modules, lockout thresholds) applies to sudo prompts too.

## Points to Remember

- `su` = switch identity for a whole session, authenticated with the **target's** credentials; `sudo` = run one command as another identity, authenticated with **your own** credentials, and logged per-invocation.
- `su -` resets environment to the target user's login environment; plain `su` keeps yours — prefer `su -` unless you specifically want to retain your own environment.
- Always edit sudoers via `visudo` (or `visudo -f` for `sudoers.d` files) — it syntax-checks before saving and prevents a lockout.
- `%groupname` in sudoers is how the `sudo`/`wheel` group mechanism works — group membership is just a rule match, not something magical baked into the OS.
- `NOPASSWD:` should be scoped to specific commands for automation, never `ALL` — that's an unaudited root shortcut if the account is ever compromised.

## Common Mistakes

- Editing `/etc/sudoers` with `vim`/`nano` directly instead of `visudo`, introducing a syntax error that locks everyone out of `sudo` (recoverable only via single-user mode or a rescue console).
- Granting `NOPASSWD: ALL` to a deploy/service account "just to make CI work," turning that account's compromise into instant root with no password barrier and often thin logging review.
- Confusing `su` session logging with `sudo`'s per-command audit trail — assuming an `su` session gives the same level of accountability that `sudo` does; it doesn't log each subsequent command the way sudo logs each invocation.
- Adding a user to the `sudo`/`wheel` group as a lazy default instead of writing a narrowly scoped sudoers rule for the specific commands they actually need.
- Using unqualified command names (`systemctl` instead of `/usr/bin/systemctl`) in a sudoers rule, which is fragile and depends on `PATH` resolution at the time of the check.
