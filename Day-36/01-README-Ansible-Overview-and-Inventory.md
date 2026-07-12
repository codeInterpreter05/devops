# Day 36 — Ansible Fundamentals: Overview & Inventory

**Phase:** 1 – Core DevOps | **Week:** W6 | **Domain:** Ansible

## Brief

Ansible is the tool most teams reach for when they need to configure servers, push consistent state across a fleet, or orchestrate a multi-step deployment — without installing agents everywhere. Where Terraform's job is "make this infrastructure exist" (provisioning), Ansible's job is "make this infrastructure look like this" (configuration management and orchestration) on machines that already exist. Interviewers ask about it constantly because most real companies still run *some* fleet of VMs/EC2 instances that need day-2 configuration, and "how do you know which hosts to run against and how do you keep them in sync" is exactly what inventory answers. Today is the on-ramp: what Ansible actually is, how it decides which hosts to talk to (inventory), and how to run one-off commands against them before you ever write a playbook.

This day is split into three files:

1. **This file** — what Ansible is, its execution model, and inventory (static + dynamic).
2. **[02-README-Playbooks-and-Ad-Hoc-Commands.md](02-README-Playbooks-and-Ad-Hoc-Commands.md)** — ad-hoc commands and playbook structure.
3. **[03-README-Variables-Facts-and-Precedence.md](03-README-Variables-Facts-and-Precedence.md)** — variables, `group_vars`/`host_vars`, and facts.

## What Ansible actually is

Ansible is an **agentless, push-based** configuration management tool written in Python. "Agentless" means there's no daemon running on the managed nodes waiting for instructions — Ansible connects over **SSH** (or WinRM for Windows), copies small Python scripts (called **modules**) to the target, executes them, and removes them. This is the core architectural difference from Puppet/Chef, which are agent-based and **pull**-based: agents on each node periodically check in with a central server and pull their desired state. Ansible instead **pushes** from a control node (your laptop, a CI runner, an Ansible Tower/AWX server) out to targets, on demand.

Why this matters practically:
- **No agent to install, upgrade, or which can itself drift or fail** — the only requirement on a target is Python and SSH access. This is why Ansible is popular for bootstrapping brand-new EC2 instances: cloud-init drops the SSH key, and Ansible can configure the box on its very first boot with zero pre-installed software beyond Python.
- **Push model means nothing runs unless you trigger it.** There's no continuous reconciliation loop like Puppet's default 30-minute pull cycle or Kubernetes controllers. Ansible runs are point-in-time: you run a playbook, it converges the target to the desired state *at that moment*, then does nothing until you run it again. This is a double-edged sword — no config drift is silently "fixed" for you overnight, but also nothing runs unexpectedly at 3am.
- **Idempotency is a design contract for module authors, not automatic.** A well-written module (e.g., `ansible.builtin.package`, `ansible.builtin.template`) checks current state before acting and reports `changed: false` if nothing needed to change. A poorly written `shell`/`command` task that just runs `useradd bob` will fail (non-idempotent) the second time you run it, because the user already exists. This is the single most common cause of "why did my playbook fail on re-run" bugs.

## Control node vs. managed nodes

- **Control node**: where you run `ansible`/`ansible-playbook` from. Requires Python 3 and the `ansible` package. Cannot be Windows natively (WSL works).
- **Managed nodes**: the targets. Need SSH (or WinRM) reachability and a Python interpreter (`ansible_python_interpreter` if it's in a non-default location, common on minimal container base images or newer Debian/Ubuntu with only `python3` and no `python` symlink).

## Inventory — telling Ansible which hosts exist

Inventory is the list of hosts (and groups of hosts) Ansible can operate against. Every `ansible`/`ansible-playbook` invocation needs one, either via `-i <path>` or a default configured in `ansible.cfg`.

### Static inventory (INI format)

```ini
# inventory.ini
[web]
web1.example.com
web2.example.com ansible_host=10.0.1.15

[db]
db1.example.com ansible_user=ec2-user ansible_ssh_private_key_file=~/.ssh/db.pem

[staging:children]
web
db

[web:vars]
http_port=8080
```

Key points:
- `[web]`, `[db]` are **groups**; hosts can belong to multiple groups.
- `[staging:children]` creates a **group of groups** — `staging` now includes every host in `web` and `db`.
- Per-host connection variables (`ansible_host`, `ansible_user`, `ansible_ssh_private_key_file`, `ansible_port`) let you override how Ansible connects to a specific host without touching SSH config files.
- `[web:vars]` sets a variable for every host in the `web` group — this is the lowest-precedence, simplest place to put a group variable (though `group_vars/web.yml`, covered in file 3, is the more maintainable convention for anything beyond a quick test).
- YAML inventory format also exists and is arguably more readable for nested groups, but INI remains the most common in the wild and in interviews.

### Dynamic inventory — the AWS EC2 plugin

Static inventory breaks down the moment your fleet auto-scales or instances get replaced — you'd be hand-editing a file every time an ASG cycles an instance. **Dynamic inventory** solves this by querying a live source (a cloud API, a CMDB) at run time to build the host list.

The modern approach is the `amazon.aws.aws_ec2` **inventory plugin** (the old standalone `ec2.py` script is deprecated):

```yaml
# inventory.aws_ec2.yml
plugin: amazon.aws.aws_ec2
regions:
  - ap-south-1
filters:
  tag:Environment: production
  instance-state-name: running
keyed_groups:
  - key: tags.Role
    prefix: role
  - key: placement.availability_zone
    prefix: az
hostnames:
  - tag:Name
  - private-ip-address
compose:
  ansible_host: public_ip_address
```

Run it with:

```bash
ansible-inventory -i inventory.aws_ec2.yml --graph
ansible all -i inventory.aws_ec2.yml -m ping
```

What's happening: the plugin calls the EC2 `DescribeInstances` API (using your normal AWS credential chain — env vars, `~/.aws/credentials`, or an instance profile if run from an EC2 box), filters instances by tag/state, and dynamically builds groups. `keyed_groups` auto-creates groups like `role_web`, `role_db`, `az_ap-south-1a` from instance tags/attributes — so a playbook can target `hosts: role_web` and it always reflects the *current* fleet, no manual file edits, no stale entries for terminated instances.

This is exactly why the interview question "static vs dynamic inventory" matters: static inventory is fine for a handful of long-lived, hand-managed servers; dynamic inventory is mandatory the moment infrastructure is elastic (ASGs, spot fleets, ephemeral CI runners), because the *source of truth* becomes the cloud provider's tags, not a file a human edits.

## Ad-hoc pings and connectivity checks

Before running anything real, always confirm reachability:

```bash
ansible all -i inventory.ini -m ping
ansible web -i inventory.ini -m ping -u ec2-user --private-key ~/.ssh/web.pem
```

The `ping` module doesn't ICMP-ping — it verifies Ansible can SSH in, execute a Python module, and get a well-formed response back. A `pong` result means the whole execution chain (SSH → Python interpreter → module execution → JSON result) works end to end.

## Points to Remember

- Ansible is agentless and push-based (SSH + temporary Python module execution); Puppet/Chef are agent-based and pull-based (periodic reconciliation). This is the #1 conceptual distinction interviewers probe.
- Idempotency is a property of well-written modules, not a guarantee of Ansible itself — raw `shell`/`command` tasks are not automatically idempotent.
- Static inventory (INI/YAML file) is fine for fixed fleets; dynamic inventory (e.g., `amazon.aws.aws_ec2` plugin) is required once infrastructure is elastic (ASGs, spot instances) so the host list always reflects live cloud state.
- `[group:children]` builds a group of groups; `[group:vars]` sets variables scoped to a group directly in the inventory file (though `group_vars/` files, covered next file, scale better).
- `ansible <group> -m ping` is the standard first command to validate SSH + Python connectivity before trusting anything else.

## Common Mistakes

- Treating a static inventory file as the source of truth for an auto-scaling fleet — it silently goes stale the moment instances are replaced, and playbooks either fail against dead IPs or miss new instances entirely.
- Forgetting that managed nodes need a Python interpreter, and getting a cryptic failure on minimal container images or `python3`-only distros; the fix is usually setting `ansible_python_interpreter=/usr/bin/python3` for that host/group.
- Assuming a `shell:` or `command:` task is idempotent just because "the playbook ran fine" — it ran fine the first time; re-running it can error out or duplicate state (e.g., appending the same line to a file twice) because there was no built-in "check current state first" logic like proper modules have.
- Hardcoding SSH usernames/keys inline in ad-hoc commands and playbooks instead of inventory `ansible_user`/`ansible_ssh_private_key_file` vars or an `ansible.cfg` default — makes the same playbook impossible to reuse across environments with different credentials.
- Confusing the AWS EC2 dynamic inventory plugin's IAM permission needs — it needs at minimum `ec2:DescribeInstances`; forgetting this produces an opaque "no hosts matched" instead of an access-denied error, because failures are often swallowed silently.
