# Day 37 — Resources: Ansible Roles & Vault

## Primary (assigned)

- **Molecule documentation** (ansible.readthedocs.io/projects/molecule) — the assigned starting point. Covers scenarios, drivers, and the full test lifecycle used in today's lab.

## Deepen your understanding

- **Ansible official docs — Roles** (docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html) — the canonical reference for role directory conventions, dependency resolution, and `include_role` vs `import_role` semantics.
- **Ansible official docs — Ansible Vault** — full command reference, `--vault-id` multi-vault workflows, and how encrypted variables interact with variable precedence.
- **Jeff Geerling's published Galaxy roles** (galaxy.ansible.com/geerlingguy) — read a few of his `tasks/main.yml` and `meta/main.yml` files as real-world examples of well-structured, cross-platform roles.

## Reference / lookup

- **Testinfra documentation** (testinfra.readthedocs.io) — the full assertion API used in Molecule `verify.yml` files (packages, services, files, sockets, processes).
- **`ansible-galaxy` CLI reference** — full flag list for `init`, `install`, `search`, `collection`.

## Practice

- Convert any playbook you've written before (or the Day 36 nginx playbook) into a role and get a green `molecule test` run across two different OS platform images — this is the single most useful practice rep for solidifying today's material.
