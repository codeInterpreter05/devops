# Day 37 — Ansible Roles & Vault: Testing Roles with Molecule

**Phase:** 1 – Core DevOps | **Week:** W6 | **Domain:** Ansible

## Brief

"How do you test Ansible roles?" is a near-guaranteed interview question, and the honest answer for any team that takes Ansible seriously is **Molecule**. Without automated testing, the only way to know a role works is to run it against a real (or staging) server and eyeball the result — slow, manual, and it doesn't scale as roles grow more conditional logic and more supported platforms. Molecule turns "does this role actually do what it claims, idempotently, across the OS versions we support" into something you can run in CI on every commit, the same way you'd unit-test application code.

## What Molecule actually does

Molecule is a testing framework purpose-built for Ansible roles. For a given role, it:

1. **Creates** one or more ephemeral test instances (via a **driver** — Docker containers are by far the most common for speed and cost, but Vagrant, cloud VMs, and others are supported).
2. **Converges** — runs your role against those instances, exactly like a real `ansible-playbook` run.
3. **Verifies** — runs assertions against the converged instances (with **Testinfra** or **Ansible's own `assert`/`Inspec`**) to confirm the role actually produced the expected state (package installed, service running, file has the right content/permissions).
4. **Idempotence check** — runs the role a *second* time and fails the test if anything reports `changed` on that second run, directly enforcing the idempotency principle from Day 36.
5. **Destroys** the test instances, leaving no residue.

## Molecule directory layout

```
roles/webserver/
  molecule/
    default/
      molecule.yml       # driver, platforms, test sequence config
      converge.yml       # the playbook that applies the role under test
      verify.yml         # Testinfra/assertions
  tasks/
  ...
```

### `molecule.yml`

```yaml
dependency:
  name: galaxy
driver:
  name: docker
platforms:
  - name: ubuntu-jammy
    image: geerlingguy/docker-ubuntu2204-ansible
    pre_build_image: true
  - name: rhel9
    image: geerlingguy/docker-rockylinux9-ansible
    pre_build_image: true
provisioner:
  name: ansible
verifier:
  name: testinfra
```

Testing against **multiple platform images in one run** is exactly how you catch the classic "works on Ubuntu, breaks on RHEL" bug (different package names, different service manager quirks) *before* it reaches a real server.

### `converge.yml`

```yaml
- name: Converge
  hosts: all
  become: true
  roles:
    - role: webserver
```

### `verify.yml` (Testinfra, Python-based)

```python
def test_nginx_installed(host):
    pkg = host.package("nginx")
    assert pkg.is_installed

def test_nginx_running_and_enabled(host):
    svc = host.service("nginx")
    assert svc.is_running
    assert svc.is_enabled

def test_config_file(host):
    f = host.file("/etc/nginx/nginx.conf")
    assert f.exists
    assert f.user == "root"
    assert f.mode == 0o644
```

## The Molecule workflow

```bash
pip install molecule molecule-plugins[docker] ansible testinfra pytest
cd roles/webserver
molecule init scenario -d docker      # scaffold if starting fresh
molecule test                         # full sequence: create -> converge -> idempotence -> verify -> destroy
molecule converge                     # just apply the role (keep instances up for iterating)
molecule verify                       # just run the verify step against instances left up
molecule idempotence                  # explicitly re-run and check for changed=0
molecule destroy                      # tear down test instances
molecule login                        # shell into a running test instance to debug interactively
```

The typical local dev loop while writing a role: `molecule converge` repeatedly (fast iteration, instances stay up), `molecule login` to poke around when something looks wrong, then `molecule test` once as the final full check before committing — the same "fast inner loop, full check before push" pattern used in most testing workflows.

## Why the idempotence check specifically matters

This is the check most unique to infrastructure automation (application unit tests don't have an equivalent). A role that reports `changed` every single run is a smell that indicates either:
- A `command`/`shell` task standing in for a proper idempotent module.
- A `template`/`lineinfile` task whose content generation isn't stable (e.g., embeds a timestamp, or the template output format doesn't exactly match what's already on disk byte-for-byte, causing a false "changed" diff every run).
- File permissions/ownership being reset in a way that doesn't match what was actually requested (fighting with another process or role).

Catching this in CI, on every PR, before the role ever touches a real server, is Molecule's single highest-value contribution — it converts "we assume this role is idempotent" into "we have proof this role is idempotent, checked automatically on every change."

## CI integration

```yaml
# .github/workflows/molecule.yml (conceptual)
jobs:
  molecule:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install molecule molecule-plugins[docker] ansible testinfra
      - run: molecule test
        working-directory: roles/webserver
```

Running this on every pull request that touches a role is what makes Molecule valuable in practice — a reviewer doesn't have to trust the PR author's manual testing claims; the pipeline proves the role converges cleanly and idempotently across every supported platform before merge.

## Points to Remember

- Molecule's test sequence is create → converge → idempotence → verify → destroy — the idempotence step (re-run, expect zero changes) is the one unique to infra-as-code testing.
- Docker is the default/fastest driver for Molecule test instances; test against every OS platform your role claims to support (`platforms:` in `molecule.yml`), not just one.
- Testinfra writes assertions in plain Python against the converged host's real state (packages, services, files) — distinct from the Ansible tasks that configured that state.
- `molecule converge` (fast, keeps instances up) is for iterating while writing a role; `molecule test` (full sequence, destroys at the end) is the CI-grade check.
- A role that never passes the idempotence check is a strong signal it's leaning on non-idempotent `command`/`shell` tasks or unstable template output.

## Common Mistakes

- Only testing a role against one OS/platform in `molecule.yml` when the role's `meta/main.yml` claims to support several — the untested platforms silently rot until someone hits a real failure in production.
- Treating `molecule converge` (which leaves instances running and doesn't check idempotence) as equivalent to `molecule test` — a role can converge fine while still failing the idempotence check, and only `test` catches that.
- Writing Testinfra assertions that just re-check the same values the Ansible task set (tautological tests) instead of verifying actual observable system behavior (e.g., checking the service is *reachable on its port*, not just that the config file has the right string in it).
- Not wiring Molecule into CI at all — running it only manually and inconsistently means it stops the moment the original role author moves on, and regressions slip in silently.
- Forgetting `molecule destroy` after manual debugging sessions (`molecule login`), leaving orphaned Docker containers accumulating locally or, worse, cloud test instances racking up cost if using a cloud driver.
