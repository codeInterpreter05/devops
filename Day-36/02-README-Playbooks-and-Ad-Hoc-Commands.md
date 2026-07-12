# Day 36 — Ansible Fundamentals: Playbooks & Ad-Hoc Commands

**Phase:** 1 – Core DevOps | **Week:** W6 | **Domain:** Ansible

## Brief

Ad-hoc commands are how you *poke* a fleet — one module, one line, no file to write. Playbooks are how you *declare and repeat* multi-step configuration reliably. Knowing when to reach for which, and being fluent in playbook YAML structure (plays, tasks, modules, handlers), is the day-to-day bread and butter of anyone running Ansible in production — and it's the format almost every Ansible interview question and take-home exercise is built around.

## Ad-hoc commands — one-off operations

The general form:

```bash
ansible <pattern> -i <inventory> -m <module> -a "<module args>" [-b] [--limit ...]
```

```bash
ansible web -i inventory.ini -m ping
ansible web -i inventory.ini -m command -a "uptime"
ansible web -i inventory.ini -m shell -a "systemctl status nginx | head -5"
ansible web -i inventory.ini -m copy -a "src=./nginx.conf dest=/etc/nginx/nginx.conf" -b
ansible web -i inventory.ini -m service -a "name=nginx state=restarted" -b
ansible all -i inventory.ini -m setup -a "filter=ansible_distribution*"
```

- `-m` selects the module (`command` is the default if omitted, but being explicit is good practice).
- `-a` passes module arguments as a single quoted `key=value` string.
- `-b` (`--become`) escalates privileges (`sudo` by default) — required for anything that touches system files or services.
- `command` vs `shell`: `command` runs the binary directly without invoking a shell — no pipes, redirects, or environment variable expansion, but *safer* (no shell injection risk from unsanitized input). `shell` runs through `/bin/sh`, so pipes/redirects/globbing work, at the cost of shell-injection risk if arguments include untrusted input. Default to `command`; reach for `shell` only when you actually need shell features.
- `-m setup` dumps **facts** (see file 3) — useful for ad-hoc discovery, e.g. "what OS/kernel is this fleet actually running."

Ad-hoc commands are for **imperative, throwaway, or diagnostic** actions — restart a service everywhere right now, check disk space across the fleet, copy one file out urgently. They are not meant to be your configuration management strategy; nothing about them is version-controlled, repeatable by someone else, or reviewable in a pull request. That's what playbooks are for.

## Playbook structure

A playbook is a YAML file containing one or more **plays**. Each play maps a set of hosts to an ordered list of **tasks**.

```yaml
---
- name: Configure web servers
  hosts: web
  become: true
  vars:
    http_port: 80
    app_name: myapp

  tasks:
    - name: Install nginx
      ansible.builtin.package:
        name: nginx
        state: present

    - name: Deploy nginx config from template
      ansible.builtin.template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/nginx.conf
        owner: root
        group: root
        mode: "0644"
      notify: Restart nginx

    - name: Ensure nginx is enabled and running
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true

  handlers:
    - name: Restart nginx
      ansible.builtin.service:
        name: nginx
        state: restarted
```

Anatomy, piece by piece:

- **`hosts:`** — which inventory group/host pattern this play targets. `hosts: web`, `hosts: all`, `hosts: web:&staging` (intersection), `hosts: web:!db` (exclusion).
- **`become: true`** — privilege escalation for the whole play (equivalent to `-b` on the CLI); can also be set per-task.
- **`vars:`** — play-scoped variables, one of many places variables can live (see file 3 for the full precedence chain).
- **`tasks:`** — the ordered list of actions. Each task calls exactly one **module** (`ansible.builtin.package`, `ansible.builtin.template`, `ansible.builtin.service`, etc. — the fully-qualified collection name (FQCN) form is the modern best practice over bare `package:`/`template:`, because it's unambiguous about which collection provides the module).
- **`notify:`** — flags a **handler** to run, but only *if the task reported `changed: true`*, and only *once* at the end of the play even if multiple tasks notify the same handler. This is the mechanism behind "restart nginx only if its config actually changed" — critical for avoiding unnecessary service restarts (which cause brief downtime) on every single run.
- **`handlers:`** — tasks that only run when notified, and run in the order they're *defined* in the handlers list (not the order they were notified), after all regular tasks in the play complete (by default — `meta: flush_handlers` can force earlier execution).

## Running a playbook

```bash
ansible-playbook -i inventory.ini site.yml
ansible-playbook -i inventory.ini site.yml --limit web1.example.com
ansible-playbook -i inventory.ini site.yml --check --diff     # dry run, show what would change
ansible-playbook -i inventory.ini site.yml --tags nginx        # run only tagged tasks (file 37 covers tags)
ansible-playbook -i inventory.ini site.yml -v                  # -v, -vv, -vvv for increasing verbosity
```

`--check` mode ("dry run") asks each module to report what *would* change without actually changing it — invaluable before running against production, though it's only as reliable as the modules involved: some modules (especially `command`/`shell`) can't meaningfully predict their effect and will just skip or warn in check mode.

## `ansible-lint` — catching mistakes before they run

`ansible-lint` statically analyzes playbooks/roles for anti-patterns: using `shell` where a proper module exists, missing `name:` on tasks, hardcoded credentials, tasks that aren't idempotent-safe, deprecated syntax. Running it in CI before merging playbook changes catches the same class of mistakes a senior reviewer would flag by hand — worth wiring into any pipeline that manages Ansible content, exactly the way `terraform validate`/`tflint` gets wired in for Terraform.

```bash
ansible-lint site.yml
ansible-lint roles/webserver/
```

## Points to Remember

- Ad-hoc commands: fast, imperative, unversioned, good for diagnostics/emergency actions. Playbooks: declarative, version-controlled, repeatable, the actual configuration management artifact.
- `command` is the safer default over `shell` (no shell interpretation/injection risk); use `shell` only when you need pipes/redirects/globbing.
- `notify` + `handlers` = the pattern for "only restart the service if config actually changed" — handlers fire once, at end of play, only on `changed: true`.
- `--check --diff` gives you a dry run with a preview of exact changes — always run this against anything production-adjacent before the real run.
- Use fully-qualified collection names (`ansible.builtin.service` not `service`) in new playbooks — avoids ambiguity as more collections get installed.

## Common Mistakes

- Using `shell`/`command` for things that have a dedicated idempotent module (e.g., `shell: apt-get install -y nginx` instead of `ansible.builtin.apt: name=nginx state=present`) — loses idempotency, proper `changed`/`ok` reporting, and check-mode support for no benefit.
- Forgetting `notify` fires handlers only on `changed: true` — then wondering why a service never restarts after a config deploy, when the actual bug is the deploy task itself isn't reporting `changed` (e.g., because file permissions/content genuinely didn't change, or because a `command` task was used instead of `template`/`copy` which track content diffs properly).
- Assuming `--check` mode is 100% trustworthy for every module — `command`/`shell` tasks generally can't be checked meaningfully and will either skip or just claim "would run," giving false confidence.
- Writing tasks without a `name:` field — makes playbook output ("TASK [...]") unreadable and debugging painful when something fails deep in a long run.
- Running a full playbook against `hosts: all` in production without `--limit` first on a single canary host, turning a small mistake in one task into a fleet-wide incident.
