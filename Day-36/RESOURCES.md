# Day 36 — Resources: Ansible Fundamentals

## Primary (assigned)

- **Ansible for DevOps** (Jeff Geerling, free to read online at ansiblefordevops.com) — the assigned starting point. Practical, example-driven, written by one of the most prolific Ansible role authors in the community.

## Deepen your understanding

- **Official Ansible documentation — Inventory Guide** (docs.ansible.com) — the canonical reference for static/dynamic inventory, group/host vars, and the `amazon.aws.aws_ec2` plugin's full option set.
- **Official Ansible documentation — Variable precedence** — the full 22-level precedence table; worth skimming once so you know it exists even though you only need the top ~6 levels memorized.
- **`ansible-doc <module>`** — run this locally, e.g. `ansible-doc ansible.builtin.template`; every module's full parameter list and examples, no internet required.

## Reference / lookup

- **Ansible Galaxy** (galaxy.ansible.com) — browse community collections and modules, including `amazon.aws` used in today's dynamic inventory lab.
- **`ansible-lint` rules documentation** — the list of anti-patterns it checks for, useful as a "what does a reviewer actually look for" checklist.

## Practice

- **Jeff Geerling's `ansible-for-devops` GitHub repo** — every example from the book as runnable code, including EC2-targeted playbooks.
- Spin up 3 free-tier EC2 instances and complete today's lab end to end — reading about idempotency is not the same as watching a re-run report `changed=0` yourself.
