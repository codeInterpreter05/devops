# Day 17 — YAML / JSON / Config Parsing: Jinja2 Templating

**Phase:** 0 – Foundation | **Week:** W3 | **Domain:** Python DevOps | **Flag:** —

## Brief

The moment you need the "same" Kubernetes Deployment for five services, or the "same" nginx config for three environments with two values different, you have a choice: copy-paste and maintain N near-identical files by hand, or generate them from one template plus a small data file. Every mature deploy tool has converged on the second answer — Ansible renders Jinja2 templates for config files on remote hosts, Helm renders Go templates for Kubernetes manifests, Terraform has its own templating via `templatefile()`. Jinja2 is the Python-native member of that family, and understanding it deeply gives you a transferable mental model for all of them: **a template describes structure with placeholders; a data file (often YAML) supplies the values; a render step combines the two into the final artifact.** Today's hands-on activity — a Jinja2-based Kubernetes manifest generator — is a hand-rolled, minimal version of what Helm does at scale.

## Core syntax

Jinja2 has three delimiter families:

```jinja2
{{ variable }}              {# expression — outputs a value #}
{% if condition %}...{% endif %}   {# statement — control flow, no output #}
{# a comment — stripped entirely from output #}
```

**Variables and attribute/item access:**

```jinja2
{{ service_name }}
{{ config.replicas }}        {# attribute access on an object, OR #}
{{ config['replicas'] }}     {# item access on a dict — Jinja2 tries both automatically #}
{{ items[0] }}                {# index access #}
```

**Loops:**

```jinja2
{% for key, value in env.items() %}
- name: {{ key }}
  value: {{ value }}
{% endfor %}
```

Inside a loop, the special `loop` variable exposes `loop.index` (1-based), `loop.index0` (0-based), `loop.first`, `loop.last`, and `loop.length` — useful for things like omitting a trailing comma on the last item of a flow-style list.

**Conditionals:**

```jinja2
{% if environment == "production" %}
replicas: {{ replicas }}
{% else %}
replicas: 1
{% endif %}
```

**Filters** transform a value with the pipe syntax, `{{ value | filter }}`, and chain left to right:

| Filter | Effect |
|---|---|
| `default('x')` | Fallback if the variable is undefined or `none` |
| `upper` / `lower` | Case conversion |
| `trim` | Strip leading/trailing whitespace |
| `join(", ")` | Join a list into a string |
| `length` | Length of a string/list/dict |
| `int` / `float` | Type coercion |
| `replace("a", "b")` | String substitution |
| `indent(4)` | Indent every line by N spaces (crucial for nesting rendered blocks inside YAML) |
| `tojson` | Serialize a Python value as a JSON literal — the safe way to embed a string/number/bool/list into a template without hand-quoting it |
| `selectattr` / `map` | Filter/transform a list of objects (e.g., `services | selectattr("enabled") | map(attribute="name") | list`) |

```jinja2
image: {{ image | default("nginx:latest") }}
name: {{ service_name | lower | replace("_", "-") }}
value: {{ raw_value | tojson }}
```

`| tojson` deserves special attention for config generation: instead of writing `value: "{{ value }}"` and hoping `value` never contains a character that breaks YAML/JSON syntax (an embedded quote, a colon, a newline), `| tojson` renders the value through Python's `json.dumps` — correctly quoted and escaped, valid YAML because YAML is a JSON superset. This is the templating equivalent of a parameterized SQL query: don't hand-build the syntax by string-gluing variable data into it — let a serializer that understands the target format's escaping rules do it.

## Whitespace control

Jinja2's `{% %}` block tags leave behind the newline and indentation of the line they're on unless told otherwise, which is a constant annoyance when generating whitespace-sensitive output like YAML:

```jinja2
{% for item in items -%}
{{ item }}
{%- endfor %}
```

The `-` inside a delimiter (`{%-` or `-%}`) strips adjacent whitespace on that side. In practice, most projects set this globally instead of sprinkling `-` everywhere:

```python
env = Environment(loader=FileSystemLoader("templates"), trim_blocks=True, lstrip_blocks=True)
```

`trim_blocks=True` removes the newline immediately after a block tag; `lstrip_blocks=True` strips leading whitespace before a block tag on its own line. Together they let you write templates with normal Python-style indentation for readability without that indentation leaking into the rendered output as stray blank lines or spaces — which matters enormously for YAML, where stray whitespace can silently change nesting.

## Template inheritance and reuse

For a family of related templates (a Deployment and a Service that share metadata conventions, or three environments' worth of a config file), Jinja2 supports **inheritance** (`extends`/`block`), **inclusion** (`include`), and **macros** (parameterized, reusable snippets — the closest thing Jinja2 has to a function).

**Inheritance** — a base template defines the skeleton and named blocks; children override specific blocks:

```jinja2
{# base_manifest.yaml.j2 #}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ service_name }}
  labels:
    app: {{ service_name }}
    env: {{ environment }}
spec:
{% block spec %}
  replicas: 1
{% endblock %}
```

```jinja2
{# production_deployment.yaml.j2 #}
{% extends "base_manifest.yaml.j2" %}
{% block spec %}
  replicas: {{ replicas }}
  strategy:
    type: RollingUpdate
{% endblock %}
```

**Macros** — reusable, parameterized fragments, imported like functions:

```jinja2
{# macros.j2 #}
{% macro container_env(env) -%}
{% for key, value in env.items() %}
- name: {{ key }}
  value: {{ value | tojson }}
{% endfor %}
{%- endmacro %}
```

```jinja2
{% import "macros.j2" as macros %}
          env:
{{ macros.container_env(env) | indent(12) }}
```

Macros are how you avoid re-deriving the same "render a list of env vars as Kubernetes `env:` entries" logic in every manifest template — write it once, `import` it wherever it's needed, exactly the same motivation as a shared Ansible role or a Helm named template (`{{ define }}`/`{{ include }}` in Helm's Go-template dialect, conceptually identical).

## Building a Kubernetes manifest generator

This is the actual pattern behind the assigned hands-on lab and, at a larger scale, behind Ansible's `template` module and (with a different template engine) Helm.

`templates/deployment.yaml.j2`:

```jinja2
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ service_name }}
  namespace: {{ namespace | default('default') }}
  labels:
    app: {{ service_name }}
    env: {{ environment }}
spec:
  replicas: {{ replicas }}
  selector:
    matchLabels:
      app: {{ service_name }}
  template:
    metadata:
      labels:
        app: {{ service_name }}
        env: {{ environment }}
    spec:
      containers:
        - name: {{ service_name }}
          image: {{ image }}
          ports:
            - containerPort: {{ container_port }}
          env:
{% for key, value in env.items() %}
            - name: {{ key }}
              value: {{ value | tojson }}
{% endfor %}
          resources:
            requests:
              cpu: {{ resources.cpu | tojson }}
              memory: {{ resources.memory | tojson }}
            limits:
              cpu: {{ (resources.cpu_limit | default(resources.cpu)) | tojson }}
              memory: {{ (resources.memory_limit | default(resources.memory)) | tojson }}
```

`render.py`:

```python
import sys
import yaml
from jinja2 import Environment, FileSystemLoader, StrictUndefined

env = Environment(
    loader=FileSystemLoader("templates"),
    trim_blocks=True,
    lstrip_blocks=True,
    undefined=StrictUndefined,   # raise immediately on any undefined variable
)

config = yaml.safe_load(open(sys.argv[1]))       # e.g. config/staging.yaml
template = env.get_template("deployment.yaml.j2")
rendered = template.render(**config)

# Fail fast if the render produced something that isn't even well-formed YAML
manifest = yaml.safe_load(rendered)

print(rendered)
```

Fed a `config/staging.yaml` of:

```yaml
service_name: payments-api
namespace: payments
environment: staging
replicas: 2
image: registry.example.com/payments-api:1.4.2
container_port: 8080
resources:
  cpu: "250m"
  memory: "256Mi"
env:
  LOG_LEVEL: debug
  FEATURE_NEW_CHECKOUT: "true"
```

this renders a complete, syntactically valid Deployment manifest with every value substituted correctly (verified: the rendered `env:` entries come out as `value: "debug"` and `value: "true"` — properly quoted strings via `| tojson`, not bare `true`/`debug` that a naive substitution or a downstream YAML re-parse could mangle).

**Why `undefined=StrictUndefined` matters:** Jinja2's default `Undefined` renders a missing variable as an **empty string**, silently. Delete `replicas` from the config with the default `Undefined` and the template renders `replicas: ` (empty) — which then either fails YAML parsing in a confusing way or, worse, parses as `null` and gets rejected three layers deeper by the Kubernetes API with an error that doesn't mention your template at all. `StrictUndefined` makes Jinja2 raise `UndefinedError` (`'replicas' is undefined`) the instant the render happens — the same "fail at the boundary" discipline as schema-validating config on load (file 1 of today).

## The Helm/Ansible connection

**Ansible** uses Jinja2 directly and natively — both for `{{ variable }}` substitution inside playbook YAML itself, and for the `template` module, which renders a `.j2` file (an nginx config, a systemd unit, an application properties file) using host/group variables as context, then copies the rendered result to the target machine. It's the identical mental model as the manifest generator above: template file + data (inventory/group_vars/host_vars) → rendered config.

**Helm** uses Go's `text/template` engine with the Sprig function library — a genuinely different templating engine, not Jinja2, with different control-flow syntax (`{{ if .Values.foo }}...{{ end }}` instead of `{% if %}...{% endif %}`) — but the same architecture: `templates/*.yaml` files with `{{ .Values.replicaCount }}`-style placeholders, a `values.yaml` supplying the data, `helm template`/`helm install` performing the render. Anywhere you see "a directory of template files plus one values file produces N environment-specific manifests," you're looking at this exact pattern, regardless of which template language implements it.

## Security note: templates are code, not strings to concatenate

Jinja2's `Template(source_string)` compiles `source_string` as executable template logic — expressions inside `{{ }}` are evaluated, and Jinja2's expression language can reach fairly deep into Python's object model. If `source_string` itself is built by concatenating **untrusted input** (a user-submitted "custom template," a value pulled from a request body) rather than being a static file you authored, you've built a Server-Side Template Injection vector (CWE-1336/CWE-94) — the input can potentially break out of being "just data" and execute template-language logic. The manifest generator above is safe by construction: the `.j2` files are static, developer-authored, loaded via `FileSystemLoader`, and only the **data** passed to `.render()` comes from the YAML config — never the template source itself. Keep it that way: if you're ever tempted to build a template string dynamically from a field in a config file, stop — pass that field in as a `render()` variable instead.

## Points to Remember

- Three delimiters: `{{ }}` expressions, `{% %}` statements, `{# #}` comments (stripped from output).
- Filters chain left-to-right with `|`; `default()`, `tojson`, and `indent()` are the three you'll reach for constantly when generating config.
- `trim_blocks=True` + `lstrip_blocks=True` on the `Environment` eliminate the stray blank lines/whitespace that block tags otherwise leave behind — essential for whitespace-sensitive output like YAML.
- `undefined=StrictUndefined` turns a silently-empty-string missing variable into an immediate `UndefinedError` at render time — fail fast instead of shipping a manifest with a blank field.
- Template inheritance (`extends`/`block`) is for full-skeleton reuse with overridable sections; macros (`{% macro %}` + `import`) are for reusable, parameterized fragments — pick inheritance for "same shape, different details," macros for "same snippet, used in several places."
- The pattern — template files (trusted, static) + data file (YAML) + render step — is identical across Jinja2/Ansible and Helm/Go-templates; only the engine differs.
- Never build template **source** from untrusted/dynamic input (`Template(user_supplied_string)`); always keep templates static and pass variable data through `.render(**data)`.

## Common Mistakes

- Using the default `Undefined` class and letting a missing/misspelled variable silently render as an empty string instead of failing the build — use `StrictUndefined` in anything that generates config someone will deploy.
- Interpolating a raw string value directly (`value: {{ value }}`) instead of `{{ value | tojson }}` — a value containing a colon, quote, or the literal words `yes`/`no`/`true` can either break the resulting YAML's structure or get silently boolean/null-coerced downstream (see file 1's Norway problem).
- Forgetting `trim_blocks`/`lstrip_blocks` (or manual `-` whitespace control) and ending up with rendered YAML full of stray blank lines — usually harmless, but occasionally enough to shift indentation and break nesting.
- Not validating the rendered output is well-formed YAML (`yaml.safe_load(rendered)`) before writing it to disk or piping it to `kubectl apply` — a template can render "successfully" (no Jinja2 error) while still producing structurally invalid or wrong YAML.
- Treating Helm's chart templates and Jinja2 as interchangeable knowledge because they look superficially similar — they're different engines with different syntax; know which one a given tool actually uses before reaching for Jinja2-specific documentation to debug a Helm chart.
- Building template **source** strings from user- or config-supplied text (a "customizable template" feature that concatenates strings before compiling) instead of passing that data through `render()` — a server-side template injection vector, not just a style issue.
