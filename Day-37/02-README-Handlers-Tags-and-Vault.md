# Day 37 — Ansible Roles & Vault: Handlers, Tags & Vault

**Phase:** 1 – Core DevOps | **Week:** W6 | **Domain:** Ansible

## Brief

Three practical mechanics separate a "toy playbook" from something you'd trust in production: **handlers** that restart services only when needed (not on every run), **tags** that let you run a *slice* of a large playbook instead of the whole thing, and **Vault** that lets secrets live safely inside version-controlled playbooks instead of sitting in plaintext or a separate, easily-forgotten system. All three come up in almost every real Ansible codebase and in almost every Ansible interview.

## Handlers, revisited inside roles

Handlers work the same inside a role (`handlers/main.yml`) as in a bare playbook (Day 36, file 2), with one added nuance: a role's handlers are visible to `notify:` calls from that role's own tasks by default, and by convention, cross-role notification (a task in role A notifying a handler defined in role B) works too, because Ansible flattens all handlers into a single namespace per play — but relying on that across roles is fragile and considered poor practice. The robust pattern is: each role owns and notifies only its own handlers.

```yaml
# roles/webserver/tasks/main.yml
- name: Deploy nginx config
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  notify: Restart nginx

# roles/webserver/handlers/main.yml
- name: Restart nginx
  ansible.builtin.service:
    name: nginx
    state: restarted
```

Handlers run **once**, at the **end of the play** (after all roles' tasks complete), in the order they're *defined*, not the order in which they were notified — and only if actually notified by a `changed: true` task. If you need a handler to fire immediately mid-play (e.g., restart a service before a later task depends on it being up), insert `meta: flush_handlers`:

```yaml
- name: Restart nginx now
  ansible.builtin.meta: flush_handlers
```

## Tags — running a slice of a playbook

Large playbooks/roles take a long time to run end to end. Tags let you label tasks and selectively execute (or skip) subsets:

```yaml
- name: Install nginx
  ansible.builtin.package:
    name: nginx
    state: present
  tags: [install]

- name: Deploy config
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  tags: [config]

- name: Ensure running
  ansible.builtin.service:
    name: nginx
    state: started
  tags: [install, config]
```

```bash
ansible-playbook site.yml --tags config          # only tasks tagged "config"
ansible-playbook site.yml --skip-tags install     # everything except "install"
ansible-playbook site.yml --tags "config,install" # union
ansible-playbook --list-tags site.yml             # discover what tags exist without running anything
```

Two special reserved tags: `always` (task runs every time regardless of `--tags` filtering, unless explicitly skipped) and `never` (task is skipped by default unless explicitly requested with `--tags never`) — useful for a debug/teardown task you never want triggered by accident. Tags can also be applied to an entire role invocation (`roles: [{role: webserver, tags: [webserver]}]`) or an entire `import_playbook`/block, cascading to every task inside.

The practical value: a config-only change (e.g., tweaking an nginx setting) doesn't need to re-run package installation and every other slow task — `--tags config` gets you a fast, targeted re-run. This is also how a lot of teams structure "day-2 operations" playbooks: one big playbook, tagged by concern, run selectively depending on what actually needs to change.

## Ansible Vault — encrypting secrets at rest

Playbooks/roles frequently need real secrets (DB passwords, API keys, TLS private keys). Committing those in plaintext to git is the single most common real-world Ansible security mistake. **Vault** encrypts values (or entire files) with AES256, using a password you supply at run time — the ciphertext is what actually lives in your git repo.

```bash
ansible-vault create secrets.yml            # create a new encrypted file, opens $EDITOR
ansible-vault edit secrets.yml              # decrypt, open in $EDITOR, re-encrypt on save
ansible-vault view secrets.yml              # decrypt to stdout, don't save
ansible-vault encrypt existing_plaintext.yml # encrypt a file you already wrote in plaintext
ansible-vault decrypt secrets.yml           # permanently decrypt (careful — don't commit this)
ansible-vault rekey secrets.yml             # change the vault password
```

Encrypted file on disk looks like this (safe to commit):

```
$ANSIBLE_VAULT;1.1;AES256
66386439653236336462626566653063336164663966303231363934653561363864363833653...
```

### Encrypting a single variable, not a whole file

Sometimes you want only one secret value encrypted inline, keeping the rest of the YAML readable and diffable in PRs:

```bash
ansible-vault encrypt_string 'S3cr3tP@ss' --name 'db_password'
```

```yaml
db_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  386264653661326439653...
db_user: admin   # stays plaintext and readable
```

### Supplying the vault password at run time

```bash
ansible-playbook site.yml --ask-vault-pass                     # interactive prompt
ansible-playbook site.yml --vault-password-file ~/.vault_pass    # from a file (chmod 600, never commit it)
ansible-playbook site.yml --vault-password-file get_vault_pass.sh  # from a script (e.g., fetches from AWS Secrets Manager/Vault itself)
```

Using a script as the "password file" is the standard CI pattern: the script fetches the actual vault password from a real secrets backend (AWS Secrets Manager, HashiCorp Vault — see Day 38) at run time rather than storing it as a static CI secret, though a well-protected CI secret variable is also acceptable in many setups.

### Multiple vault IDs — different passwords per environment

```bash
ansible-vault encrypt --vault-id prod@prompt group_vars/production/vault.yml
ansible-vault encrypt --vault-id staging@prompt group_vars/staging/vault.yml
ansible-playbook site.yml --vault-id prod@~/.vault_pass_prod --vault-id staging@~/.vault_pass_staging
```

`--vault-id` namespaces secrets by label so a staging engineer's laptop doesn't need (and can't accidentally use) the production vault password — a meaningful blast-radius reduction over one shared vault password for everything.

## Points to Remember

- Handlers fire once, at the end of the play, in definition order, only on `changed: true`; use `meta: flush_handlers` to force an earlier run when ordering actually matters.
- Tags let you run/skip a labeled subset of tasks (`--tags`, `--skip-tags`, `--list-tags`); `always` and `never` are reserved special tags.
- Ansible Vault encrypts files or individual string values with AES256 so secrets can live safely in git as ciphertext; `encrypt_string` keeps diffs readable by encrypting only the sensitive value, not the whole file.
- Never commit the vault password itself — supply it via `--ask-vault-pass` (interactive), a gitignored password file, or a script that fetches it from a real secrets backend.
- `--vault-id` supports multiple named vault passwords (e.g., per environment), reducing the blast radius of a single leaked password.

## Common Mistakes

- Committing the vault password file itself into the same repo as the encrypted secrets it protects — defeats the entire point of encryption.
- Encrypting an entire `group_vars/production.yml` file (mixing secrets with non-secret config) instead of using `encrypt_string` for just the sensitive keys — makes PR review impossible because reviewers can't see what non-secret values actually changed inside an opaque encrypted blob.
- Forgetting that handlers only fire on `changed: true` and then debugging the wrong task when a service fails to restart — the actual bug is often that the *triggering* task isn't correctly detecting a change (e.g., a `command` task used where `template`/`copy` would properly report `changed`).
- Using one single vault password for every environment (dev, staging, prod) — anyone who can decrypt the dev secrets file can decrypt production secrets, since it's the same password.
- Assuming `--tags` selects tasks package-wide when a role itself wasn't given matching tags — tags on individual tasks inside a role only take effect if the role invocation (or the tasks themselves) actually carries that tag; a common gotcha is tagging tasks inside a role but forgetting older Ansible versions required propagating tags to the role import itself.
