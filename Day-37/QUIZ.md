# Day 37 — Quiz: Ansible Roles & Vault

Try to answer without looking at your notes. Answers are at the bottom.

1. What problem do roles solve that a single flat playbook doesn't?
2. What is the difference in precedence and intent between a role's `defaults/main.yml` and `vars/main.yml`?
3. When you list a role under a play's `roles:` key alongside a `tasks:` section, which runs first — and how do you control that if you need a different order?
4. What does `meta/main.yml`'s `dependencies:` key do?
5. What is `requirements.yml` for, and why shouldn't you vendor downloaded Galaxy role contents directly into your own repo's version control?
6. What do the reserved tags `always` and `never` do?
7. Why is `ansible-vault encrypt_string` often preferable to `ansible-vault encrypt` on a whole file?
8. What are the 5 stages of a `molecule test` run, and which one specifically checks idempotency?
9. Why does Molecule typically test against multiple platform images rather than just one?
10. What's the practical difference between `molecule converge` and `molecule test`, and when would you use each?
11. **Interview question:** How do you test Ansible roles? What is Molecule and what does it test?

---

## Answers

1. Roles let you decompose a playbook into named, independently reusable, independently testable units (like functions/libraries) instead of one long flat task list — the same role (e.g., a monitoring agent) can be applied across multiple unrelated host groups without duplicating logic.
2. `defaults/main.yml` has the *lowest* precedence of any variable source and is meant to be freely overridden by callers (sensible defaults). `vars/main.yml` has *high* precedence and is meant for role-internal constants that callers generally shouldn't (and structurally can't easily) override.
3. `roles:` always executes before the play's own `tasks:`, regardless of their order in the YAML file. To interleave a role at a specific point relative to other tasks, use `include_role`/`import_role` directly inside the `tasks:` list instead of the top-level `roles:` key.
4. It declares other roles this role depends on; Ansible resolves and automatically runs those dependency roles (in listed order) before the depending role's own tasks execute.
5. `requirements.yml` pins exact versions of external roles/collections your project depends on (analogous to a lockfile), and `ansible-galaxy install -r requirements.yml` fetches them reproducibly in any environment/CI run. Vendoring downloaded role source directly into your repo bloats it, makes upstream updates harder to track, and duplicates content that's already versioned and hosted on Galaxy.
6. `always` makes a task run every time regardless of `--tags`/`--skip-tags` filtering (unless explicitly skipped via `--skip-tags always`). `never` makes a task skipped by default, running only if explicitly requested with `--tags never` (or another tag it also carries) — useful for dangerous/debug tasks you never want to fire accidentally.
7. `encrypt_string` encrypts only the specific sensitive value inline, leaving the rest of the YAML file plaintext and readable/diffable in pull requests. Encrypting an entire file makes the whole thing an opaque blob, so reviewers can't see what non-secret configuration actually changed in a PR.
8. Create, converge, idempotence, verify, destroy. The **idempotence** stage specifically re-runs the role a second time and fails the test if anything reports `changed`, directly validating the idempotency principle.
9. Because roles frequently behave differently across OS families (different package names, package managers, service manager quirks) — testing only one platform lets platform-specific bugs (e.g., a role that works on Ubuntu but breaks on RHEL) slip through undetected until they hit a real server.
10. `molecule converge` just applies the role and leaves the test instances running — useful for fast iteration while actively writing/debugging a role, since you can immediately `molecule login` to inspect state. `molecule test` runs the full sequence including idempotence and verify checks and then destroys the instances — this is the CI-grade, complete check you'd run before merging.
11. Strong answer: "We test Ansible roles with Molecule, a framework that spins up ephemeral test instances (usually Docker containers) for one or more target platforms, applies (`converges`) the role against them exactly like a real run, then re-runs it a second time to assert idempotency — nothing should report `changed` on the second pass. After that it runs a `verify` step using Testinfra (or similar) to assert the actual resulting system state — packages installed, services running and enabled, file permissions correct — then tears the instances down. We wire `molecule test` into CI so every PR that touches a role proves it converges cleanly and idempotently across every OS platform we claim to support, instead of relying on manual testing against a real server."
