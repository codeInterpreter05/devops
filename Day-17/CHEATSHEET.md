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
yaml.load(f, Loader=yaml.UnsafeLoader)   # or Loader=yaml.Loader — executes !!python/object/apply:... tags
yaml.unsafe_load(f)                       # same risk, shorthand form
# yaml.load(f) with no Loader= now raises TypeError (PyYAML 5.1+) instead of silently defaulting to unsafe
# FullLoader blocks the code-exec tags but safe_load() is still the only recommended choice
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
from jinja2 import Environment, FileSystemLoader, StrictUndefined

env = Environment(
    loader=FileSystemLoader("templates/"),
    trim_blocks=True,       # eat the newline after a {% block %} tag
    lstrip_blocks=True,     # eat leading whitespace before a {% block %} tag
    undefined=StrictUndefined,   # raise UndefinedError on any missing variable, don't render ""
)
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
from pydantic import SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="APP_", env_file=".env", env_nested_delimiter="__")

    database_url: str
    log_level: str = "INFO"
    max_retries: int = 3
    api_key: SecretStr           # repr/str always show '**********', never the real value

settings = Settings()             # raises ValidationError if required vars missing/wrong type
settings.api_key.get_secret_value()   # only way to get the real value back out
```

## Base + overlay merge (avoid duplicating YAML per environment)

```python
from copy import deepcopy

def deep_merge(base: dict, override: dict) -> dict:
    result = deepcopy(base)
    for k, v in override.items():
        if isinstance(v, dict) and isinstance(result.get(k), dict):
            result[k] = deep_merge(result[k], v)
        else:
            result[k] = v
    return result
# {**base, **override} is WRONG here — it replaces nested dicts wholesale instead of merging them
```

## Config drift detection

```bash
terraform plan                 # non-empty diff on unchanged .tf = drift
driftctl scan                  # detects cloud resources unmanaged/drifted from Terraform state
kubectl diff -f manifest.yaml   # shows what `apply` would change, without applying it
argocd app diff <app>          # GitOps: live cluster state vs. git (Argo CD marks "OutOfSync")
ansible-playbook site.yml --check --diff   # dry-run: reports drift from playbook's declared state
aws configservice get-compliance-details-by-config-rule --config-rule-name <rule>
```
