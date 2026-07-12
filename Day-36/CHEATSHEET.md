# Day 36 — Cheatsheet: Ansible Fundamentals

## Ad-hoc commands

```bash
ansible all -i inventory.ini -m ping                     # connectivity check
ansible web -i inventory.ini -m command -a "uptime"       # run a command
ansible web -i inventory.ini -m shell -a "df -h | grep sda"   # needs shell features (pipes)
ansible web -i inventory.ini -m copy -a "src=f dest=/tmp/f" -b
ansible web -i inventory.ini -m service -a "name=nginx state=restarted" -b
ansible all -i inventory.ini -m setup                     # dump all facts
ansible all -i inventory.ini -m setup -a "filter=ansible_distribution*"
ansible web -i inventory.ini -m package -a "name=nginx state=present" -b
```

## Inventory (static, INI)

```ini
[web]
web1 ansible_host=10.0.1.10 ansible_user=ec2-user

[db]
db1 ansible_host=10.0.1.20

[prod:children]
web
db

[web:vars]
http_port=80
```

## Inventory (dynamic, AWS EC2 plugin)

```yaml
# inventory.aws_ec2.yml
plugin: amazon.aws.aws_ec2
regions: [ap-south-1]
filters:
  tag:Environment: production
  instance-state-name: running
keyed_groups:
  - key: tags.Role
    prefix: role
hostnames: [tag:Name]
compose:
  ansible_host: public_ip_address
```

```bash
ansible-galaxy collection install amazon.aws
ansible-inventory -i inventory.aws_ec2.yml --graph
ansible role_web -i inventory.aws_ec2.yml -m ping
```

## Playbook skeleton

```yaml
---
- name: Configure web servers
  hosts: web
  become: true
  vars:
    http_port: 80
  tasks:
    - name: Install nginx
      ansible.builtin.package:
        name: nginx
        state: present
    - name: Deploy config
      ansible.builtin.template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: Restart nginx
  handlers:
    - name: Restart nginx
      ansible.builtin.service:
        name: nginx
        state: restarted
```

## Running playbooks

```bash
ansible-playbook -i inventory.ini site.yml
ansible-playbook -i inventory.ini site.yml --limit web1
ansible-playbook -i inventory.ini site.yml --check --diff    # dry run + diff
ansible-playbook -i inventory.ini site.yml --tags nginx
ansible-playbook -i inventory.ini site.yml -e "app_version=1.4.2"
ansible-playbook -i inventory.ini site.yml -vvv              # max verbosity
```

## Host pattern syntax

```
all                # every host
web                # group
web:db             # union
web:&prod          # intersection
web:!staging       # exclusion
web1.example.com   # single host
'*.example.com'    # wildcard (quote to avoid shell globbing)
```

## Variable precedence (low -> high)

```
role defaults/main.yml
inventory vars (group_vars/, host_vars/, [group:vars])
playbook vars: / vars_files:
role vars/main.yml
task-level vars:
-e / --extra-vars (CLI, always wins)
```

## Facts

```yaml
"{{ ansible_facts.os_family }}"           # RedHat / Debian
"{{ ansible_facts.distribution }}"        # Ubuntu / CentOS
"{{ ansible_facts.default_ipv4.address }}"
```

```yaml
- hosts: web
  gather_facts: false   # skip if play doesn't need facts (faster runs)
```

## Linting

```bash
pip install ansible-lint
ansible-lint site.yml
ansible-lint roles/webserver/
```
