# Day 37 — Ansible Roles & Vault: Roles & Galaxy

**Phase:** 1 – Core DevOps | **Week:** W6 | **Domain:** Ansible

## Brief

A single giant `site.yml` works for a demo; it falls apart the moment two playbooks need "install and configure nginx" or three teams need slightly different variants of "set up a database server." **Roles** are Ansible's unit of reuse — a standardized directory layout that packages tasks, variables, templates, handlers, and files into something you can drop into any playbook, share across projects, or publish for others via **Galaxy**. Interviewers ask about roles constantly because "how do you structure Ansible content at scale" separates people who've only run a tutorial playbook from people who've maintained Ansible in a real, multi-team codebase.

## Why roles exist

Without roles, a playbook doing several unrelated things (web server, database, monitoring agent) accumulates one flat, sprawling `tasks:` list. Roles let you decompose that into named, independently testable, independently reusable units:

```yaml
# site.yml — orchestration only, no logic
- hosts: web
  roles:
    - webserver
    - monitoring-agent

- hosts: db
  roles:
    - postgresql
    - monitoring-agent
```

`monitoring-agent` appears once but is used against two completely different host groups — write it once, use it everywhere. This is the direct analog of a function or a shared library in general software engineering: the playbook becomes an orchestration layer, and the role holds the actual implementation.

## Standard role directory structure

```
roles/
  webserver/
    tasks/
      main.yml          # the role's actual task list — entry point
    handlers/
      main.yml          # handlers scoped to this role
    templates/
      nginx.conf.j2      # Jinja2 templates, referenced via template module
    files/
      favicon.ico        # static files, referenced via copy module
    vars/
      main.yml           # high-precedence, role-internal constants
    defaults/
      main.yml           # low-precedence, user-overridable defaults
    meta/
      main.yml           # role metadata: dependencies, supported platforms, Galaxy info
    tests/
      test.yml           # a minimal playbook that exercises the role (pre-Molecule convention)
    README.md
```

Ansible auto-discovers each of these by convention — you never explicitly say "load `tasks/main.yml`"; simply naming the role in a playbook's `roles:` list wires up every directory automatically, in this order: `defaults` loaded first (lowest precedence, see Day 36 file 3), then `vars`, then `tasks/main.yml` executed, with `handlers` available to anything that notifies them and `meta/main.yml` dependencies pulled in *before* the role's own tasks run.

### `meta/main.yml` — dependencies and metadata

```yaml
galaxy_info:
  role_name: webserver
  author: your_name
  description: Installs and configures nginx
  license: MIT
  min_ansible_version: "2.14"
  platforms:
    - name: Ubuntu
      versions: [focal, jammy]
    - name: EL
      versions: [8, 9]

dependencies:
  - role: common
    vars:
      some_var: value
```

`dependencies:` lets one role require another to run first (e.g., a `webserver` role depending on a `common` role that sets up base packages/users) — Ansible resolves and runs dependencies automatically before the depending role's own tasks, in the order listed, with duplicate-dependency handling to avoid running the same shared role twice per play unless explicitly allowed.

### Using a role in a playbook — two syntaxes

```yaml
# Classic (runs before all explicit "tasks:" in the play)
- hosts: web
  roles:
    - webserver
    - { role: monitoring-agent, agent_port: 9100 }

# Modern (import_role/include_role — runs inline, wherever placed in tasks:)
- hosts: web
  tasks:
    - name: Apply webserver role
      ansible.builtin.include_role:
        name: webserver
```

The classic `roles:` list always executes *before* any tasks defined directly in the play's `tasks:` section, regardless of where `roles:` is positioned in the YAML — a common source of "why did my role run before the pre-task I wrote above it" confusion. `include_role`/`import_role` inside `tasks:` avoids this by running exactly where you place it in the sequence. `import_role` is resolved statically at parse time (like a compile-time include); `include_role` is resolved dynamically at run time (supports looping over roles, conditional role application) — the same static-vs-dynamic distinction that applies to `import_tasks` vs `include_tasks`.

## Ansible Galaxy — the community role/collection registry

`ansible-galaxy` is both the CLI tool and the public registry (galaxy.ansible.com) for sharing roles and collections.

```bash
ansible-galaxy install geerlingguy.nginx              # install a published role
ansible-galaxy install -r requirements.yml            # install everything pinned in a requirements file
ansible-galaxy init roles/mynewrole                   # scaffold a new role with the standard directory layout
ansible-galaxy collection install amazon.aws           # install a collection (modules + roles + plugins bundled)
```

`requirements.yml` pins exact versions for reproducibility — exactly like a `package-lock.json` or `Pipfile.lock`:

```yaml
roles:
  - name: geerlingguy.nginx
    version: "3.1.4"

collections:
  - name: amazon.aws
    version: ">=6.0.0"
  - name: community.general
```

Committing `requirements.yml` (not the downloaded role contents themselves — those go in `.gitignore`, typically under `roles/` or wherever `ansible.cfg`'s `roles_path` points) and running `ansible-galaxy install -r requirements.yml` in CI is the standard pattern — it keeps external role code out of your own repo while guaranteeing every environment installs the exact same pinned version.

## Points to Remember

- Roles are the reuse boundary in Ansible: a self-contained package of tasks/handlers/templates/vars/defaults, wired together purely by directory convention.
- `defaults/main.yml` = low precedence, meant to be overridden by callers; `vars/main.yml` = high precedence, role-internal constants.
- Playbook-level `roles:` always runs before that play's own `tasks:`, regardless of YAML ordering; use `include_role`/`import_role` inside `tasks:` when execution order matters.
- `meta/main.yml`'s `dependencies:` lets roles require other roles, resolved and run automatically before the dependent role.
- `ansible-galaxy` + `requirements.yml` is how you pull in and pin community roles/collections without vendoring their source into your own repo.

## Common Mistakes

- Putting environment-specific or secret values into a role's `defaults/main.yml` and shipping it that way — defaults are meant to be safe, generic fallbacks, not a place to leak real hostnames or credentials.
- Expecting a role placed under `roles:` to execute in the exact YAML position relative to `pre_tasks`/`tasks` — it always runs before `tasks:` (though `pre_tasks` genuinely does run before `roles`), which surprises people used to top-to-bottom execution.
- Not pinning role/collection versions in `requirements.yml`, then having a previously-working playbook silently break because an upstream Galaxy role changed behavior between runs.
- Reinventing a role from scratch that already exists, well-maintained, on Galaxy (e.g., `geerlingguy.nginx`, `geerlingguy.mysql`) instead of composing/wrapping it — burns time and misses edge cases the community role already handles.
- Confusing `vars/main.yml` (role-internal, high precedence, hard to override) with `defaults/main.yml` (user-facing, low precedence, meant to be overridden) and being surprised a "configurable" value set in `vars/` can't be changed by the caller.
