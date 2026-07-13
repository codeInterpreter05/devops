# Day 112 — Lab: Platform Engineering Concepts

**Goal:** Stand up a real Backstage instance, register your own GitHub repos in its Software Catalog, wire up TechDocs for one of them, and build a Scaffolder template that produces a fully catalog-registered new Python service.

**Prerequisites:** Node.js 18+, Yarn, Docker (for Postgres), a GitHub account with at least one or two existing repos you can register, and a GitHub OAuth App or personal access token for catalog discovery.

```bash
npx @backstage/create-app@latest --path backstage-lab
cd backstage-lab
```

---

### Lab 1 — Run Backstage locally and add your first catalog entity

1. Start the app in dev mode (uses SQLite by default — fine for this lab):
   ```bash
   yarn dev
   ```
   This opens the frontend at `http://localhost:3000` and the backend at `http://localhost:7007`.
2. In one of your own GitHub repos, add a `catalog-info.yaml` at the repo root:
   ```yaml
   apiVersion: backstage.io/v1alpha1
   kind: Component
   metadata:
     name: my-first-service
     description: A service registered for the platform engineering lab
     annotations:
       github.com/project-slug: <your-github-username>/<repo-name>
   spec:
     type: service
     lifecycle: experimental
     owner: guests
   ```
   Commit and push it.
3. In the Backstage UI, go to **Create** → **Register Existing Component**, and paste the raw GitHub URL to your `catalog-info.yaml`.
4. Confirm the entity appears in the catalog with the correct owner and lifecycle badge.

**Success criteria:** Your repo shows up as a `Component` in the Backstage catalog UI, with metadata pulled directly from the YAML you committed — not manually typed into a form.

---

### Lab 2 — Wire up TechDocs for that same repo

1. Add a `docs/index.md` and `mkdocs.yml` to the repo root:
   ```yaml
   # mkdocs.yml
   site_name: 'my-first-service'
   nav:
     - Home: index.md
   plugins:
     - techdocs-core
   ```
2. Add the TechDocs annotation to `catalog-info.yaml`:
   ```yaml
   metadata:
     annotations:
       backstage.io/techdocs-ref: dir:.
   ```
3. Commit, push, and in the Backstage UI open the entity's **Docs** tab — trigger a docs build if it doesn't refresh automatically.

**Success criteria:** Your Markdown renders as a docs page inside the entity's own Backstage page, not in a separate wiki tab.

---

### Lab 3 — The core hands-on activity: create a software template for new Python services

This is the assigned hands-on activity for today — build the golden path, don't just read about it.

1. Create a minimal skeleton directory in a new `templates/python-service` repo (or folder), containing a bare Python service skeleton (`app.py`, `requirements.txt`, a `Dockerfile`, a GitHub Actions CI workflow) with Nunjucks-style placeholders for the values you'll collect (e.g. `${{ values.name }}`).
2. Write the template definition:
   ```yaml
   apiVersion: scaffolder.backstage.io/v1beta3
   kind: Template
   metadata:
     name: python-service-template
     title: New Python Microservice
     description: Scaffolds a Python service with CI and catalog registration
   spec:
     owner: guests
     type: service
     parameters:
     - title: Service details
       required: [name, owner]
       properties:
         name:
           title: Service name
           type: string
         owner:
           title: Owner
           type: string
     steps:
     - id: fetch
       name: Fetch skeleton
       action: fetch:template
       input:
         url: ./skeleton
         values:
           name: ${{ parameters.name }}
           owner: ${{ parameters.owner }}
     - id: publish
       name: Publish
       action: publish:github
       input:
         repoUrl: github.com?owner=<your-github-username>&repo=${{ parameters.name }}
     - id: register
       name: Register
       action: catalog:register
       input:
         repoContentsUrl: ${{ steps.publish.output.repoContentsUrl }}
         catalogInfoPath: /catalog-info.yaml
     output:
       links:
       - title: Repository
         url: ${{ steps.publish.output.remoteUrl }}
   ```
3. Register the template itself in the catalog (same "Register Existing Component" flow as Lab 1, pointing at the template YAML).
4. Run it from **Create** → pick your new template → fill the form → submit.
5. Confirm: a brand-new GitHub repo was created with the skeleton content, and it shows up in the catalog automatically — with no manual catalog PR from you.

**Success criteria:** You can point to a new repo and its catalog entry, both created purely by filling out a form — the fetch → publish → register pipeline actually ran end to end.

---

### Lab 4 — Explore ownership and dependency graphs

1. Register a second entity (a `System` grouping both services), and set `spec.system` on both `Component` entities to point at it.
2. In the catalog UI, open the System's page and confirm both components show up as members.
3. Add a `providesApis`/`consumesApis` relationship between the two components (even a placeholder API entity) and confirm the dependency graph renders it visually.

**Success criteria:** You can navigate from a System to its member Components and see a rendered dependency graph, not just a flat list.

---

### Cleanup

```bash
# Stop the dev server (Ctrl+C), then optionally remove the local app
rm -rf backstage-lab
# Delete any test repos created via the Scaffolder template from your GitHub account
```

### Stretch challenge

Add a fourth Scaffolder step that calls a webhook/GitHub Action to also provision a Crossplane `PostgreSQLInstance` Claim (see Day 113) for the new service as part of scaffolding — describe exactly which step type and inputs you'd use, even if you don't have a live Crossplane cluster to actually run it against.
