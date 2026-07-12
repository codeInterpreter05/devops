# Day 17 — Lab: YAML / JSON / Config Parsing

**Goal:** Build a working Jinja2-based Kubernetes manifest generator driven by YAML config, while exercising YAML gotchas, JSON Schema validation, and typed environment config along the way.

**Prerequisites:** Python 3.11+, a virtual environment (see Day 15 lab if you need a refresher), and:
```bash
python3 -m venv .venv && source .venv/bin/activate
pip install pyyaml ruamel.yaml jsonschema jinja2 pydantic pydantic-settings python-dotenv
```

---

### Lab 1 — Reproduce the YAML "Norway problem" yourself

1. Create `gotcha.yaml`:
   ```yaml
   country: NO
   enabled: yes
   version: 1.20
   ```
2. Load it and inspect types:
   ```python
   import yaml
   data = yaml.safe_load(open("gotcha.yaml"))
   for k, v in data.items():
       print(k, repr(v), type(v))
   ```
3. Observe that `country` becomes `False` (bool), `enabled` becomes `True`, and `version` becomes `1.2` (float, trailing zero dropped) — not the strings you probably expected.
4. Fix the file by quoting the ambiguous values (`country: "NO"`, `version: "1.20"`) and re-run — confirm they now come back as strings.

**Success criteria:** You can explain, without checking notes, why `country: NO` becomes `False` under YAML 1.1, and you've fixed it with quoting.

---

### Lab 2 — Multi-document YAML and `safe_load` vs `safe_load_all`

1. Create `multi.yaml`:
   ```yaml
   apiVersion: v1
   kind: Deployment
   metadata:
     name: app-a
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: app-a-svc
   ```
2. Run `yaml.safe_load(open("multi.yaml"))` and observe it raises `yaml.composer.ComposerError: expected a single document in the stream` — it does not silently return the first document, it refuses to parse at all.
3. Fix it with `list(yaml.safe_load_all(open("multi.yaml")))` and confirm both documents come back.

**Success criteria:** You can state from memory that `safe_load()` on a multi-document stream raises `ComposerError` (a loud failure, not a silent one) and that `safe_load_all()` is what you need instead.

---

### Lab 3 — JSON Schema validation

1. Write a schema for an application config (name: string, replicas: int >= 1, env: enum of dev/staging/prod, `additionalProperties: False`).
2. Validate three test configs against it: one valid, one missing a required field, one with an extra/typo'd key (e.g., `replias` instead of `replicas`).
3. Confirm the schema catches both the missing-field case and the typo'd-key case (the second only works if `additionalProperties` is set to `False` — try it both ways and observe the difference).

**Success criteria:** You have a working validate-then-raise pattern, and you can explain what `additionalProperties: False` catches that a schema without it would miss.

---

### Lab 4 — The core hands-on activity: Jinja2 Kubernetes manifest generator

This is the assigned hands-on activity — do it end to end.

1. Create `templates/deployment.yaml.j2`:
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: {{ app_name }}
     labels:
       app: {{ app_name }}
   spec:
     replicas: {{ replicas }}
     selector:
       matchLabels:
         app: {{ app_name }}
     template:
       metadata:
         labels:
           app: {{ app_name }}
       spec:
         containers:
           - name: {{ app_name }}
             image: {{ image }}
             ports:
               - containerPort: {{ port | default(8080) }}
             env:
   {% for key, value in env_vars.items() %}
             - name: {{ key }}
               value: {{ value | tojson }}
   {% endfor %}
   ```
   Note the `| tojson` on the env value, not manual quotes (`"{{ value }}"`) — `tojson` correctly escapes/quotes whatever comes out of the YAML config, including values that could otherwise be misread as booleans (see Lab 1's Norway problem — an env var value of `DEBUG: on` must render as the literal string `"on"`, not get coerced).
2. Create `configs/dev.yaml`:
   ```yaml
   app_name: diskcheck
   replicas: 1
   image: myregistry/diskcheck:dev
   env_vars:
     LOG_LEVEL: DEBUG
   ```
   and `configs/prod.yaml` with `replicas: 5`, `image: myregistry/diskcheck:v1.4.2`, `LOG_LEVEL: INFO`.
3. Write `generate.py` that loads a config YAML and renders the template through an `Environment` configured with `trim_blocks=True, lstrip_blocks=True, undefined=StrictUndefined` — parameterize the config path via `click` (from Day 15) or plain `argparse`.
4. Run it for both `dev.yaml` and `prod.yaml`, and diff the two generated manifests — confirm only the expected fields differ.
5. Validate each generated manifest is actually parseable YAML by round-tripping it through `yaml.safe_load()` — this is the check that catches a template that "rendered" but produced broken YAML.
6. Now delete the `replicas` key from `configs/dev.yaml` and re-run the generator. Confirm it raises `jinja2.exceptions.UndefinedError: 'replicas' is undefined` immediately, rather than rendering `replicas: ` (empty) and failing later, more confusingly, at the `yaml.safe_load()` step or inside Kubernetes itself. Then try it again with a plain `Environment(loader=...)` (no `undefined=StrictUndefined`) to see the silent-empty-string behavior for comparison, and restore the key afterward.

**Success criteria:** Running the generator against both config files produces two valid, structurally-correct Kubernetes manifests differing only in the intended fields, both pass a `yaml.safe_load()` round-trip, and you've directly observed `StrictUndefined` turning a missing-variable bug into an immediate, clear error instead of a silent blank field.

---

### Lab 5 — Typed config with `pydantic-settings`

1. Define a `Settings(BaseSettings)` class with `database_url: str`, `log_level: str = "INFO"`, `max_retries: int = 3`.
2. Run your script with no environment variables set and observe the `ValidationError` for the missing required `database_url`.
3. Set `APP_DATABASE_URL` (or your chosen prefix) and re-run — confirm it now loads cleanly and `max_retries` is a real `int`, not a string.
4. Intentionally set `APP_MAX_RETRIES=notanumber` and observe pydantic reject it with a clear type-coercion error.

**Success criteria:** You can explain why failing fast with a `ValidationError` at startup is preferable to a scattered `os.environ.get()` pattern that fails later with a less obvious error.

---

### Cleanup

```bash
deactivate
rm -rf .venv gotcha.yaml multi.yaml templates/ configs/ generate.py
```

### Stretch challenge

Extend the manifest generator to also render a `Service` and a `ConfigMap` from the same config file using `{% include %}` or Jinja2 macros to avoid repeating the `metadata`/`labels` block across all three templates, and add a `base.yaml` + per-environment overlay (dev/prod) that your script deep-merges before rendering — so environment-specific config files only contain the handful of keys that actually differ.
