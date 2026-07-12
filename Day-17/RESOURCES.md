# Day 17 — Resources: YAML / JSON / Config Parsing

## Primary (assigned)

- **Jinja2 docs** (jinja.palletsprojects.com) and **Pydantic docs** (docs.pydantic.dev) — free, official, the assigned starting point for today. Jinja2's "Template Designer Documentation" page covers the templating syntax in full; Pydantic's docs cover both core validation and the `pydantic-settings` extension.

## Deepen your understanding

- **The Norway Problem** — search "YAML Norway problem" for the well-documented history of this exact bug class; useful for internalizing why quoting ambiguous scalars matters, beyond just memorizing the rule.
- **JSON Schema official docs** (json-schema.org) — the spec-level reference for everything expressible in a schema (types, enums, patterns, nested objects, `additionalProperties`, conditional schemas).
- **The Twelve-Factor App — Config** (12factor.net/config) — the canonical short read on why config should be injected via environment, not duplicated in code, underpinning today's interview question.
- **`ruamel.yaml` documentation** (yaml.readthedocs.io) — covers round-trip preservation in depth, useful once you're building tools that programmatically edit existing YAML files.

## Reference

- **PyYAML documentation** (pyyaml.org/wiki/PyYAMLDocumentation) — the safe_load/safe_load_all API surface and loader security notes.
- **Kustomize documentation** (kubectl.docs.kubernetes.io/references/kustomize) — the base+overlay pattern implemented natively for Kubernetes manifests, a production example of the pattern covered in file 3.

## Practice

- **Helm's chart template guide** (helm.sh/docs/chart_template_guide) — a different engine (Go `text/template` + Sprig, not Jinja2), but the same template-plus-values-file mental model; working through their examples reinforces the pattern in a real-world tool with different syntax to compare against.
- **jsonschema.net** — paste a sample JSON/YAML document and get a starter schema generated automatically, useful for quickly bootstrapping a schema for practice data.
