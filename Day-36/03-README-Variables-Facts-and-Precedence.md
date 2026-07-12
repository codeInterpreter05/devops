# Day 36 — Ansible Fundamentals: Variables, Facts & Precedence

**Phase:** 1 – Core DevOps | **Week:** W6 | **Domain:** Ansible

## Brief

Ansible lets you define the same variable name in at least ten different places — play `vars`, role defaults, `group_vars`, `host_vars`, command-line `-e`, facts, and more — and it needs a deterministic rule for which one wins. Not understanding **variable precedence** is the single most common source of "why is my playbook using the wrong value" confusion, and it's a favorite interview probe because it reveals whether someone has actually debugged a real Ansible codebase or just followed a tutorial once.

## Where variables can live

| Location | Scope | Typical use |
|---|---|---|
| `vars:` in a play | That play only | Quick, playbook-specific values |
| `vars_files:` | Imported into the play | Externalizing a big block of vars from the playbook file |
| `defaults/main.yml` (in a role) | Role, lowest precedence | Sensible defaults a role ships with, meant to be overridden |
| `vars/main.yml` (in a role) | Role, high precedence | Role-internal constants not meant to be overridden by users |
| `group_vars/<groupname>.yml` | All hosts in that inventory group | Environment-wide config (e.g., `group_vars/production.yml`) |
| `host_vars/<hostname>.yml` | One specific host | Per-host overrides (e.g., a unique IP or hostname-specific tuning) |
| `-e "var=value"` on CLI | Whole run, highest precedence | One-off overrides, CI-injected values |
| Registered variables (`register:`) | Play, from that point on | Capturing a task's output for later use |
| Facts (`ansible_facts`) | Play, gathered per host | Live host-discovered data (see below) |

`group_vars/` and `host_vars/` are directories Ansible auto-loads based on filename matching group/host names in your inventory — no explicit `vars_files:` include needed, which is what makes them the standard, scalable way to organize environment-specific configuration (e.g., `group_vars/all.yml` for global defaults, `group_vars/production.yml` and `group_vars/staging.yml` for environment splits).

```
inventory/
  hosts.ini
  group_vars/
    all.yml
    web.yml
    production.yml
  host_vars/
    web2.example.com.yml
```

## Precedence — the rule that actually matters

Ansible documents roughly 22 precedence levels, but the ones you actually need memorized, from **lowest to highest**:

1. Role `defaults/main.yml` (lowest — meant to be overridden)
2. Inventory file/group/host vars (`[group:vars]`, `group_vars/`, `host_vars/`)
3. Playbook `vars:`, `vars_files:`
4. Role `vars/main.yml`
5. Task-level `vars:`
6. `-e`/`--extra-vars` on the command line (**highest** — always wins, no exceptions)

The rule of thumb that covers 95% of real debugging: **more specific beats less specific, and command-line `-e` always wins no matter what.** A `host_vars` entry for one host overrides a `group_vars` entry that would otherwise apply to it; a role's own `vars/` overrides its `defaults/`; and nothing beats `-e` — which is exactly why CI pipelines use `-e` to inject environment-specific secrets/parameters at run time without touching any files.

```bash
ansible-playbook -i inventory.ini site.yml -e "app_version=1.4.2" -e "@secrets.yml"
```

(`-e @file.yml` loads variables from a file — useful for passing a whole block without a giant one-liner.)

## Ansible facts — live, host-discovered data

At the start of every play (unless disabled), Ansible runs the `setup` module on each target and gathers **facts**: OS family, distribution and version, IP addresses, mounted filesystems, CPU count, memory, hostname, and much more. These become variables under the `ansible_facts` namespace (and historically also as flat top-level vars like `ansible_distribution`, `ansible_default_ipv4`, though the namespaced form is now preferred).

```yaml
- name: Show distribution
  ansible.builtin.debug:
    msg: "{{ ansible_facts.distribution }} {{ ansible_facts.distribution_version }}"

- name: Install package differently per OS family
  ansible.builtin.package:
    name: "{{ 'httpd' if ansible_facts.os_family == 'RedHat' else 'apache2' }}"
    state: present
```

This is what makes a single playbook portable across Ubuntu and RHEL fleets — branch on `ansible_facts.os_family` (`Debian` vs `RedHat`) instead of hardcoding a package manager or package name.

Fact gathering has a real cost — it SSHes in and runs a non-trivial Python script per host, which adds latency, especially across hundreds of hosts. If a play never references any fact, disable it:

```yaml
- hosts: web
  gather_facts: false
```

Or gather a narrower subset via the `setup` module's `filter:` argument when you need speed but still want a couple of specific facts.

## Registered variables — capturing task output

```yaml
- name: Check if config file exists
  ansible.builtin.stat:
    path: /etc/myapp/config.yml
  register: config_check

- name: Create default config if missing
  ansible.builtin.copy:
    src: default-config.yml
    dest: /etc/myapp/config.yml
  when: not config_check.stat.exists
```

`register` captures the full result object (return values like `changed`, `stat`, `rc`, `stdout`) of a task into a variable usable by later tasks — the standard pattern for conditional logic based on live system state discovered mid-play.

## Points to Remember

- Precedence, low to high: role `defaults` → inventory vars (`group_vars`/`host_vars`) → playbook `vars` → role `vars` → task `vars` → `-e` (command line always wins).
- `group_vars/` and `host_vars/` are auto-loaded by filename matching — no explicit include needed, and they're the scalable way to separate environment config from playbook logic.
- Facts are gathered fresh per host per play via the implicit `setup` module run — they reflect live state, not what you last remembered about a box, which is why they're safe to branch logic on (`ansible_facts.os_family`).
- `gather_facts: false` when a play doesn't use any facts — meaningfully speeds up large-fleet runs.
- `register` + `when` is the core pattern for "check state, then act conditionally" — this is how you make playbooks that adapt instead of blindly re-applying the same steps.

## Common Mistakes

- Assuming role `defaults/` and role `vars/` behave the same way — `defaults` is intentionally the *lowest* precedence (easy to override from outside), while `vars/` is *high* precedence (hard to override) — mixing these up means a "configurable" role parameter silently can't be overridden.
- Forgetting `-e` on the CLI overrides *everything*, including values a role explicitly sets in its own `vars/` — leads to confusing "I set this in the role, why is the CLI value still winning" bugs (answer: that's `-e`'s entire purpose).
- Referencing `ansible_distribution` or other fact-derived variables in a play that has `gather_facts: false` — the variable is simply undefined, and the failure message doesn't always make the root cause obvious.
- Putting environment-specific values (prod DB hostnames, prod-only feature flags) directly in a playbook's `vars:` block instead of `group_vars/production.yml` — makes the same playbook impossible to safely run against staging without manual edits.
- Not realizing `register` captures the *entire* result object, not just stdout — new users often write `{{ config_check }}` expecting a string and get a whole dict dumped, when they wanted `{{ config_check.stdout }}` or `{{ config_check.stat.exists }}`.
