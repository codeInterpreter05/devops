# Day 37 — Lab: Ansible Roles & Vault

**Goal:** Convert yesterday's flat nginx playbook into a proper role, encrypt a real secret with Vault, and prove the role is idempotent and correct using Molecule against a Docker driver.

**Prerequisites:**
- Completed Day 36 (a working `site.yml` that installs/configures nginx).
- `pip install ansible ansible-lint molecule molecule-plugins[docker] testinfra pytest`
- Docker installed and running locally (Molecule's default driver).

---

### Lab 1 — Scaffold and populate the role

1. Scaffold the standard role layout:
   ```bash
   ansible-galaxy init roles/webserver
   ```
2. Move your Day 36 template into `roles/webserver/templates/nginx.conf.j2`.
3. Move your tasks into `roles/webserver/tasks/main.yml`, and your handler into `roles/webserver/handlers/main.yml`.
4. Set a sensible default in `roles/webserver/defaults/main.yml`:
   ```yaml
   http_port: 80
   ```
5. Rewrite `site.yml` as pure orchestration:
   ```yaml
   - name: Configure web servers
     hosts: web
     become: true
     roles:
       - webserver
   ```
6. Lint it:
   ```bash
   ansible-lint roles/webserver/ site.yml
   ```

**Success criteria:** `ansible-playbook -i inventory.ini site.yml` still installs/configures nginx correctly, now sourced entirely from the role.

---

### Lab 2 — Add tags and prove selective runs

1. Tag the install task `install`, the template/notify task `config`, and the service-enable task `install, config` inside `roles/webserver/tasks/main.yml`.
2. Run only the config slice:
   ```bash
   ansible-playbook -i inventory.ini site.yml --tags config
   ```
3. Confirm via `--list-tags`:
   ```bash
   ansible-playbook -i inventory.ini site.yml --list-tags
   ```

**Success criteria:** `--tags config` re-templates the config and restarts nginx (if changed) without re-running the package install task (visible in the play recap as skipped).

---

### Lab 3 — Encrypt a real secret with Vault

1. Add a fake "API key" the nginx status page will reference as an environment-style variable:
   ```bash
   ansible-vault encrypt_string 'sk_live_fake_12345' --name 'api_key' >> roles/webserver/vars/vault.yml
   ```
2. Reference it (harmlessly, just to prove decryption works) in a debug task at the end of `tasks/main.yml`:
   ```yaml
   - name: Confirm secret decrypts (do not do this with a real secret in real code)
     ansible.builtin.debug:
       msg: "Key starts with {{ api_key[:7] }}"
   ```
3. Store a vault password in a gitignored file:
   ```bash
   echo "supersecretvaultpw" > ~/.vault_pass
   chmod 600 ~/.vault_pass
   echo "roles/webserver/vars/vault.yml" >> .gitignore  # actually don't ignore it, DO commit the encrypted file
   ```
   Correction: you want the **encrypted file** committed (it's ciphertext, safe to commit) — only the password file itself must never be committed. Add `~/.vault_pass` style paths to `.gitignore` only if it lives inside the repo (it shouldn't).
4. Run with the vault password file supplied:
   ```bash
   ansible-playbook -i inventory.ini site.yml --vault-password-file ~/.vault_pass
   ```
5. Confirm running without `--vault-password-file` fails with a clear "vault password required" error — proving the secret really is encrypted at rest.

**Success criteria:** `cat roles/webserver/vars/vault.yml` shows unreadable `$ANSIBLE_VAULT;1.1;AES256` ciphertext, and the playbook only succeeds when the correct vault password is supplied.

---

### Lab 4 — Test the role with Molecule against Docker

1. Initialize a Molecule scenario for the role:
   ```bash
   cd roles/webserver
   molecule init scenario -d docker
   ```
2. Edit `molecule/default/molecule.yml` to test against two platforms:
   ```yaml
   platforms:
     - name: ubuntu-jammy
       image: geerlingguy/docker-ubuntu2204-ansible
       pre_build_image: true
   ```
3. Edit `molecule/default/converge.yml` to apply the role (excluding the Vault-encrypted var for simplicity, or supply a `molecule.yml`-level `provisioner.env` var pointing at a test vault password).
4. Write `molecule/default/verify.yml` (Testinfra) asserting nginx is installed, running, enabled, and the config file has mode `0644`.
5. Run the full sequence:
   ```bash
   molecule test
   ```
6. Deliberately break idempotency (temporarily replace the `template` task with a `shell: echo "x" >> /etc/nginx/nginx.conf` task) and re-run `molecule test` to watch the idempotence check fail — then revert the change.

**Success criteria:** `molecule test` passes fully (create, converge, idempotence, verify, destroy) on the correct role, and you've personally witnessed the idempotence check catch a broken, non-idempotent task.

---

### Cleanup

```bash
molecule destroy
docker ps -a | grep molecule   # confirm no leftover containers
rm ~/.vault_pass
```

### Stretch challenge

Add a second Molecule platform (e.g., a RHEL-family image) to `molecule.yml`, adjust the role's tasks to branch on `ansible_facts.os_family` for the package name/manager, and get `molecule test` passing on both platforms in the same run.
