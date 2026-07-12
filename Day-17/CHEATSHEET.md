# Day 17 — Cheatsheet: YAML / JSON / Config Parsing

## YAML gotchas quick reference

```yaml
# Ambiguous scalars YAML 1.1 coerces automatically — always quote these:
country: "NO"        # unquoted NO/no/No -> boolean False
enabled: "yes"        # unquoted yes/on/true -> boolean True
version: "1.20"       # unquoted 1.20 -> float 1.2 (trailing zero dropped)
zip_code: "08080"     # unquoted leading-zero numbers behave inconsistently across parsers

# Anchors, aliases, merge keys
defaults: &defaults
  timeout: 30
service:
  <<: *defaults
  timeout: 60          # overrides just this key

# Multi-document file
---
kind: Deployment
---
kind: Service
```

## PyYAML

```python
import yaml

yaml.safe_load(open("f.yaml"))            # single document -> dict/list
list(yaml.safe_load_all(open("f.yaml")))   # multiple --- separated documents
yaml.dump(data, default_flow_style=False)  # dict -> YAML text

# NEVER on untrusted input:
yaml.load(f, Loader=yaml.FullLoader)   # can construct arbitrary Python objects
```

## `ruamel.yaml` — round-trip preserving

```python
from ruamel.yaml import YAML

yaml = YAML()
yaml.preserve_quotes = True
data = yaml.load(open("values.yaml"))
data["replicas"] = 5
yaml.dump(data, open("values.yaml", "w"))   # comments/order/formatting preserved
```

## `jsonschema` validation

```python
from jsonschema import validate, ValidationError

schema = {
    "type": "object",
    "properties": {
        "name": {"type": "string"},
        "replicas": {"type": "integer", "minimum": 1},
        "env": {"type": "string", "enum": ["dev", "staging", "prod"]},
    },
    "required": ["name", "replicas", "env"],
    "additionalProperties": False,   # catch typo'd keys
}

try:
    validate(instance=config, schema=schema)
except ValidationError as exc:
    print(exc.path, exc.message)
```

## Jinja2

```python
from jinja2 import Environment, FileSystemLoader

env = Environment(loader=FileSystemLoader("templates/"))
template = env.get_template("deployment.yaml.j2")
output = template.render(app_name="diskcheck", replicas=3)
```

```jinja2
{{ variable }}                       {# expression: produces output #}
{% if condition %} ... {% endif %}   {# statement: control flow, no output #}
{# a comment, stripped entirely #}

{% for k, v in items.items() %}
  - {{ k }}: {{ v }}
{% endfor %}

{{ value | default("fallback") }}
{{ dict_value | tojson }}
{{ name | lower | replace(" ", "-") }}

{% extends "base.yaml.j2" %}
{% block spec %} ... {% endblock %}
{% include "snippet.yaml.j2" %}
{% macro container(name, image) %} ... {% endmacro %}
```

## `python-dotenv`

```python
from dotenv import load_dotenv
import os

load_dotenv()                 # loads .env into os.environ, local dev only
db_url = os.environ["DATABASE_URL"]
```

## `pydantic-settings`

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    database_url: str
    log_level: str = "INFO"
    max_retries: int = 3
    model_config = {"env_prefix": "APP_", "env_file": ".env"}

settings = Settings()   # raises ValidationError if required vars missing/wrong type
```

## Config drift detection

```bash
terraform plan                 # non-empty diff on unchanged .tf = drift
aws configservice get-compliance-details-by-config-rule --config-rule-name <rule>
argocd app diff <app>          # GitOps: live cluster state vs. git
```
