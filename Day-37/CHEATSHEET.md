# Day 37 — Cheatsheet: Ansible Roles & Vault

## Role directory layout

```
roles/webserver/
  tasks/main.yml       # entry point, the role's actual task list
  handlers/main.yml    # handlers scoped to this role
  templates/*.j2       # Jinja2 templates (template module)
  files/*              # static files (copy module)
  vars/main.yml        # high precedence, role-internal constants
  defaults/main.yml    # low precedence, user-overridable
  meta/main.yml        # dependencies, supported platforms, Galaxy info
```

## ansible-galaxy

```bash
ansible-galaxy init roles/mynewrole            # scaffold a role
ansible-galaxy install geerlingguy.nginx        # install a published role
ansible-galaxy install -r requirements.yml      # install pinned roles/collections
ansible-galaxy collection install amazon.aws
ansible-galaxy list                             # list installed roles
```

```yaml
# requirements.yml
roles:
  - name: geerlingguy.nginx
    version: "3.1.4"
collections:
  - name: amazon.aws
    version: ">=6.0.0"
```

## Using roles in a playbook

```yaml
- hosts: web
  roles:
    - webserver
    - { role: monitoring, agent_port: 9100 }

- hosts: web
  tasks:
    - ansible.builtin.include_role:   # dynamic, runs inline in task order
        name: webserver
    - ansible.builtin.import_role:    # static, resolved at parse time
        name: webserver
```

## Tags

```yaml
tags: [install]
tags: [always]     # always runs unless --skip-tags always
tags: [never]       # only runs if explicitly requested
```

```bash
ansible-playbook site.yml --tags config
ansible-playbook site.yml --skip-tags install
ansible-playbook site.yml --list-tags
```

## Handlers

```yaml
notify: Restart nginx      # fires only if the task reports changed: true
- ansible.builtin.meta: flush_handlers   # force pending handlers to run now
```

## Ansible Vault

```bash
ansible-vault create secrets.yml
ansible-vault edit secrets.yml
ansible-vault view secrets.yml
ansible-vault encrypt plain.yml
ansible-vault decrypt secrets.yml
ansible-vault rekey secrets.yml
ansible-vault encrypt_string 'value' --name 'var_name'
```

```bash
ansible-playbook site.yml --ask-vault-pass
ansible-playbook site.yml --vault-password-file ~/.vault_pass
ansible-playbook site.yml --vault-id prod@~/.vault_pass_prod
```

```yaml
db_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  666236396...
```

## Molecule

```bash
pip install molecule molecule-plugins[docker] ansible testinfra pytest
molecule init scenario -d docker
molecule test          # create -> converge -> idempotence -> verify -> destroy
molecule converge       # apply role only, keep instances up
molecule verify         # run verify.yml/Testinfra only
molecule idempotence    # re-run + check changed=0
molecule login          # shell into test instance
molecule destroy        # tear down
```

```yaml
# molecule.yml
driver:
  name: docker
platforms:
  - name: ubuntu-jammy
    image: geerlingguy/docker-ubuntu2204-ansible
    pre_build_image: true
provisioner:
  name: ansible
verifier:
  name: testinfra
```

```python
# verify.yml (Testinfra)
def test_nginx(host):
    assert host.package("nginx").is_installed
    assert host.service("nginx").is_running
    assert host.service("nginx").is_enabled
```
