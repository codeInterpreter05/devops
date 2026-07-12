# Day 36 — Lab: Ansible Fundamentals

**Goal:** Write and run a real playbook that configures 3 EC2 instances (or 3 local Docker/Vagrant boxes if you don't want to spend cloud money) — installing nginx, deploying a templated config, and enabling the service — while practicing both static and dynamic inventory.

**Prerequisites:**
- Ansible installed on your control machine: `pip install ansible` (or `pipx install ansible`).
- `ansible-lint` installed: `pip install ansible-lint`.
- Either: 3 EC2 instances you can SSH into (a free-tier-eligible `t3.micro` x3 works fine), or 3 local containers as stand-ins (`docker run -d --name web1 -p 2201:22 <ssh-enabled-image>` style, or 3 Vagrant/Multipass VMs). Instructions below assume EC2 but note the Docker-only alternative for each step.
- The `amazon.aws` collection for dynamic inventory: `ansible-galaxy collection install amazon.aws` and `pip install boto3 botocore`.

---

### Lab 1 — Static inventory and ad-hoc connectivity

1. Create `inventory.ini`:
   ```ini
   [web]
   web1 ansible_host=<ip1> ansible_user=ec2-user
   web2 ansible_host=<ip2> ansible_user=ec2-user
   web3 ansible_host=<ip3> ansible_user=ec2-user

   [web:vars]
   ansible_ssh_private_key_file=~/.ssh/my-key.pem
   ansible_python_interpreter=/usr/bin/python3
   ```
2. Confirm connectivity:
   ```bash
   ansible web -i inventory.ini -m ping
   ```
3. Run one ad-hoc command to check OS facts across all three:
   ```bash
   ansible web -i inventory.ini -m setup -a "filter=ansible_distribution*"
   ```

**Success criteria:** `ansible web -m ping` returns `pong` from all three hosts with no SSH errors.

---

### Lab 2 — The core hands-on activity: playbook to install and configure nginx

1. Create the project structure:
   ```bash
   mkdir -p ansible-nginx/templates
   cd ansible-nginx
   ```
2. Create `templates/nginx.conf.j2`:
   ```nginx
   events {}
   http {
     server {
       listen {{ http_port }};
       location / {
         return 200 "Hello from {{ inventory_hostname }}, managed by Ansible\n";
       }
     }
   }
   ```
3. Create `site.yml`:
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

       - name: Deploy nginx config
         ansible.builtin.template:
           src: templates/nginx.conf.j2
           dest: /etc/nginx/nginx.conf
           mode: "0644"
         notify: Restart nginx

       - name: Ensure nginx enabled and running
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
4. Lint before running:
   ```bash
   ansible-lint site.yml
   ```
5. Dry-run first:
   ```bash
   ansible-playbook -i ../inventory.ini site.yml --check --diff
   ```
6. Run for real:
   ```bash
   ansible-playbook -i ../inventory.ini site.yml
   ```
7. Verify from your control node:
   ```bash
   ansible web -i ../inventory.ini -m uri -a "url=http://{{ '{{' }} ansible_host {{ '}}' }}"
   # or simply:
   curl http://<web1-ip>/
   ```
8. Re-run the exact same playbook a second time and confirm every task reports `ok` (not `changed`) except nothing needed to change — proving idempotency.

**Success criteria:** All 3 hosts serve the Ansible-templated nginx page, and a second run of the playbook shows `changed=0` across the board.

---

### Lab 3 — Dynamic inventory with the AWS EC2 plugin

1. Tag your 3 EC2 instances with `Role=web` and `Environment=lab` in the AWS console or CLI.
2. Create `inventory.aws_ec2.yml`:
   ```yaml
   plugin: amazon.aws.aws_ec2
   regions:
     - <your-region>
   filters:
     tag:Environment: lab
     instance-state-name: running
   keyed_groups:
     - key: tags.Role
       prefix: role
   hostnames:
     - tag:Name
   compose:
     ansible_host: public_ip_address
   ```
3. Confirm AWS credentials are available (`aws sts get-caller-identity`), then inspect the generated inventory:
   ```bash
   ansible-inventory -i inventory.aws_ec2.yml --graph
   ```
4. Re-run Lab 2's playbook against the dynamic group instead of the static file:
   ```bash
   ansible-playbook -i inventory.aws_ec2.yml site.yml --limit role_web
   ```

**Success criteria:** `ansible-inventory --graph` shows a `role_web` group populated purely from EC2 tags, and the playbook runs successfully against it with zero manual host entries.

---

### Cleanup

```bash
# Terminate the lab EC2 instances via console or:
aws ec2 terminate-instances --instance-ids <id1> <id2> <id3>
rm -rf ansible-nginx inventory.ini inventory.aws_ec2.yml
```

### Stretch challenge

Modify the playbook so that on RHEL/Amazon Linux it installs `nginx` via `dnf` with EPEL considerations, and on Ubuntu it installs via `apt`, using a single unified task that branches on `ansible_facts.os_family` — without writing two separate playbooks.
