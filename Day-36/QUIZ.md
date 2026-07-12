# Day 36 — Quiz: Ansible Fundamentals

Try to answer without looking at your notes. Answers are at the bottom.

1. Is Ansible agent-based or agentless, and is it push or pull? Name one tool that is the opposite on each axis.
2. Why doesn't Ansible require any long-running daemon on managed nodes, and what does it actually need there instead?
3. What's the difference between `command` and `shell` modules, and why is `command` the safer default?
4. What does `notify:` + a `handlers:` block actually guarantee about when and how often a handler runs?
5. What problem does dynamic inventory (e.g., the `amazon.aws.aws_ec2` plugin) solve that a static inventory file cannot?
6. Rank these from lowest to highest variable precedence: task `vars:`, role `defaults/main.yml`, `-e` on the CLI, `group_vars/`.
7. What are Ansible facts, when are they gathered, and how can you speed up a large-fleet run that doesn't need them?
8. What does `--check --diff` do, and what's a class of tasks where it can't give you a fully reliable preview?
9. In an inventory file, what's the difference between `[web:vars]` and `[web:children]`?
10. Why is idempotency described as "a property of the module, not of Ansible itself"? Give an example of a non-idempotent task.
11. **Interview question:** What is the difference between Ansible and Terraform? When do you use each?

---

## Answers

1. Ansible is agentless and push-based — a control node SSHes out and pushes changes on demand. Puppet and Chef are the classic opposite: agent-based and pull-based, with an agent on each node periodically checking in with a central server.
2. Ansible transfers small Python modules over SSH, executes them on the target, and cleans up — no persistent process is needed because nothing is "listening" for work; work only happens when you explicitly run `ansible`/`ansible-playbook`. The managed node just needs SSH access and a Python interpreter.
3. `command` executes the binary directly with no shell involved — no pipes, redirects, globbing, or env var expansion, but also no shell-injection risk from untrusted input. `shell` runs through `/bin/sh`, enabling pipes/redirects, but is riskier if arguments include unsanitized data. Default to `command`; use `shell` only when you need shell-specific features.
4. `notify` schedules a handler to run only if the notifying task reports `changed: true`, and the handler runs at most once at the end of the play (by default), even if multiple tasks notify it — never once per notifying task.
5. Static inventory is a fixed file that goes stale the moment infrastructure changes (auto-scaling, instance replacement, spot termination). Dynamic inventory queries a live source (the EC2 API via tags/filters) at run time, so the host list always reflects the fleet's *current* state with zero manual editing.
6. Lowest to highest: role `defaults/main.yml` → `group_vars/` (inventory vars) → task `vars:` → `-e` on the CLI (always wins, highest of all).
7. Facts are host-discovered data (OS, IP addresses, memory, mounted filesystems, etc.) gathered via the implicit `setup` module at the start of each play (unless disabled). Set `gather_facts: false` on plays that never reference any fact to skip this SSH round-trip and speed up large runs.
8. `--check` performs a dry run — modules report what *would* change without changing it — and `--diff` shows the actual content diff for file-based changes. It's unreliable for `command`/`shell` tasks, which generally can't predict their own effect and just skip or warn in check mode.
9. `[web:vars]` sets variables that apply to every host in the `web` group directly in the inventory file. `[web:children]` (note: written as `[groupname:children]`, e.g. `[prod:children]` listing `web` and `db`) creates a group-of-groups — a parent group whose members are the *hosts of the listed child groups*, not a variable assignment.
10. Ansible itself doesn't verify state before acting — that logic lives inside each module's implementation. Well-written modules (`package`, `template`, `service`) check current state and report `changed: false` if nothing needs to happen. A `command: useradd bob` task has no such check — running it twice fails the second time because the user already exists, which is a classic non-idempotent task.
11. Terraform is a **provisioning**/infrastructure-as-code tool — its job is to create, modify, and destroy cloud resources (VPCs, EC2 instances, S3 buckets, IAM roles) and track their state so it knows what exists. Ansible is a **configuration management/orchestration** tool — its job is to configure software *on* machines that already exist (install packages, deploy configs, restart services, orchestration sequencing across a fleet). In practice they're complementary, not competing: Terraform provisions the EC2 instances, then Ansible (often triggered right after, e.g. via a Terraform `local-exec` or a separate CI stage) configures what runs on them. A red flag answer treats them as interchangeable; a strong answer explains the provisioning vs. configuration boundary and gives a concrete example of using both together.
