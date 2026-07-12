# Day 11 — Resources: Docker Deep Dive II

## Primary (assigned)

- **Docker Compose documentation** (docs.docker.com/compose) — free, the assigned starting point for this day. Covers the `compose.yaml` file reference, the Compose v2 CLI, `depends_on` health conditions, and the `deploy.resources` block used in today's lab.

## Deepen your understanding

- **Docker networking overview** (docs.docker.com/network) — the official reference for all four network drivers (bridge, host, none, overlay), including the embedded DNS behavior and why the default bridge network doesn't get it.
- **Docker storage: volumes** (docs.docker.com/storage/volumes) — the authoritative reference on named volumes, bind mounts, tmpfs, and volume drivers; clarifies exactly what Docker does and doesn't manage for each.
- **Julia Evans — "Docker networking: bridge, none, and host"** (jvns.ca) — short, illustrated, builds real intuition for the namespace/veth mechanics underneath the CLI, in the same spirit as her Linux zines referenced on Day 1.
- **Red Hat — "An introduction to Linux network routing" and cgroups background** (a search for "cgroups v2 kernel documentation" or Red Hat's cgroups explainer) — useful background on the actual kernel mechanism (`cgroups`) that Docker's `--memory`/`--cpus` flags configure; helps ground *why* an OOM kill is scoped to a container instead of the host.

## Reference / lookup

- `man 7 network_namespaces` and `man 8 ip-netns` — on-box manual pages for the Linux primitives underneath Docker's network drivers, useful during the namespace-inspection lab exercise.
- **Compose file reference** (docs.docker.com/compose/compose-file) — the definitive field-by-field spec for `services`, `networks`, `volumes`, `healthcheck`, and `deploy.resources` keys.

## Practice

- **Play with Docker** (labs.play-with-docker.com) — free, browser-based multi-node Docker playground; good for experimenting with Swarm `overlay` networks across multiple simulated hosts without provisioning real VMs.
- **Katacoda-style Docker scenarios / Docker's own "Get Started" multi-container tutorial** (docs.docker.com/get-started) — reinforces the same app+cache+db Compose pattern used in today's lab with a slightly different stack, good for a second rep.
