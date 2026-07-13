# Day 112 — Cheatsheet: Platform Engineering & Backstage

## Bootstrap a Backstage app

```bash
npx @backstage/create-app@latest --path my-backstage-app
cd my-backstage-app
yarn dev                     # frontend :3000, backend :7007
yarn build:backend            # production build
yarn workspace app add <plugin-package>   # add a frontend plugin
yarn workspace backend add <plugin-package>  # add a backend plugin
```

## catalog-info.yaml essentials

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component        # or: API, Resource, System, Domain, User, Group, Location
metadata:
  name: my-service
  description: One-line description
  annotations:
    github.com/project-slug: org/repo
    backstage.io/techdocs-ref: dir:.
  tags: [python, payments]
spec:
  type: service          # service | website | library
  lifecycle: production  # experimental | production | deprecated
  owner: team-payments
  system: checkout
  providesApis: [payments-api]
  consumesApis: [billing-api]
  dependsOn: [component:default/database]
```

## Registering entities

```
UI: Create -> Register Existing Component -> paste raw catalog-info.yaml URL
```
```yaml
# app-config.yaml — auto-discovery instead of manual registration
catalog:
  providers:
    github:
      myOrg:
        organization: 'my-org'
        catalogPath: '/catalog-info.yaml'
        schedule:
          frequency: { minutes: 30 }
          timeout: { minutes: 3 }
```

## TechDocs

```yaml
# mkdocs.yml
site_name: 'my-service'
nav:
  - Home: index.md
plugins:
  - techdocs-core
```
```yaml
# catalog-info.yaml annotation
metadata:
  annotations:
    backstage.io/techdocs-ref: dir:.
```

## Scaffolder template skeleton

```yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata: { name: my-template, title: My Template }
spec:
  owner: guests
  type: service
  parameters:
  - title: Details
    properties:
      name: { type: string }
  steps:
  - id: fetch
    action: fetch:template
    input: { url: ./skeleton, values: { name: '${{ parameters.name }}' } }
  - id: publish
    action: publish:github
    input: { repoUrl: 'github.com?owner=org&repo=${{ parameters.name }}' }
  - id: register
    action: catalog:register
    input:
      repoContentsUrl: '${{ steps.publish.output.repoContentsUrl }}'
      catalogInfoPath: /catalog-info.yaml
```

## Common built-in Scaffolder actions

```
fetch:template        # template a skeleton with parameter values
fetch:plain           # fetch static files, no templating
publish:github        # create/push to a new GitHub repo
publish:gitlab        # create/push to a new GitLab repo
catalog:register      # register the new repo's catalog-info.yaml
debug:log             # print a debug message during scaffolding
```

## Platform engineering vocabulary quick reference

```
IDP              = catalog/portal + self-service provisioning + golden paths
                   + standard CI/CD + built-in observability/security defaults
Golden path      = easiest, most-supported way to do a common task (opt-in, not forced)
Paved road       = same idea as golden path (Spotify's term)
Self-service     = provision without a human-in-the-loop ticket
Platform team    = ongoing service to stream-aligned teams (Team Topologies)
Enabling team    = temporary help adopting a new capability, then leaves
Platform-as-Product = platform team treats devs as customers, does discovery, tracks adoption
```
