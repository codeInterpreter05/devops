# Day 17 — YAML / JSON / Config Parsing: YAML & JSON Validation

**Phase:** 0 – Foundation | **Week:** W3 | **Domain:** Python DevOps | **Flag:** —

## Brief

Almost every tool you'll touch as a DevOps engineer is configured through YAML: Kubernetes manifests, Helm values, GitHub Actions/GitLab CI pipelines, Ansible playbooks, Docker Compose files. YAML's readability is also its trap — it has more implicit type coercion and syntactic ambiguity than JSON, and a misparsed config doesn't fail loudly at the point of the mistake; it fails later, often in production, as a confusing runtime error three layers removed from the typo. Interviewers probe this because it separates people who've only ever "written some YAML" from people who've been bitten by it — the boolean coercion bugs, the multi-document manifest that silently only processed the first resource, the deserialization footgun in `yaml.load`. This note covers YAML's actual parsing rules, the PyYAML/ruamel.yaml split, and how to stop trusting config blindly by validating it against a schema before it's used.

This day is split into three files:

1. **This file** — YAML syntax and gotchas, PyYAML vs ruamel.yaml, JSON Schema validation with `jsonschema`.
2. **[02-README-Jinja2-Templating.md](02-README-Jinja2-Templating.md)** — Jinja2 in depth, and using it to generate Kubernetes manifests from a YAML data source (the Helm/Ansible pattern).
3. **[03-README-Environment-Config-And-Drift.md](03-README-Environment-Config-And-Drift.md)** — dev/staging/prod config with `python-dotenv` and `pydantic-settings`, the base+overlay pattern for avoiding YAML duplication, and config drift detection.

## YAML syntax: the parts that bite

YAML (YAML Ain't Markup Language) is a superset of JSON — any valid JSON is valid YAML — built around indentation-defined structure instead of braces.

**Mappings** (dicts) and **sequences** (lists) are indentation-scoped:

```yaml
service_name: payments-api
replicas: 3
resources:
  cpu: "250m"      # nested mapping — 2 spaces deeper
  memory: "256Mi"
tags:
  - billing        # sequence items start with "- "
  - critical
```

Indentation must be **spaces only** — YAML forbids tabs for indentation, and most parsers throw a hard error rather than silently guessing. The width isn't fixed (2 vs 4 spaces both work) but it must be **consistent within a block**; mixing indent widths at the same nesting level is a parse error.

YAML also has a **flow style** that looks like JSON, and you'll see it mixed with block style constantly in real configs:

```yaml
env: {LOG_LEVEL: info, DEBUG: "false"}   # flow mapping
ports: [8080, 9090]                       # flow sequence
```

### Multi-document files

A single `.yaml` file can hold several independent documents separated by `---` — exactly how a Kubernetes manifest commonly bundles a `Deployment` and a `Service` in one file, or how a Helm chart's rendered output stacks resources. This trips people up in a specific way:

```python
import yaml

yaml.safe_load(open("multi.yaml"))       # raises yaml.composer.ComposerError:
                                          # "expected a single document in the stream"
list(yaml.safe_load_all(open("multi.yaml")))   # correct: returns a generator of every document
```

`safe_load()` on a multi-document stream doesn't silently drop the extra documents — it **raises `ComposerError`** and refuses to parse at all. That's actually the safer failure mode (loud, not silent), but it's still a common source of "this worked on my single-resource test file and blew up on the real manifest" bugs. Any code that might receive a `---`-separated file — anything reading raw Kubernetes YAML, Helm template output, or multi-service Compose-adjacent configs — should use `safe_load_all()` and iterate.

### Anchors, aliases, and merge keys

YAML lets you define a node once and reuse it — this is how tools like Docker Compose and hand-maintained CI configs avoid repeating themselves inside a single file:

```yaml
defaults: &defaults
  restart: always
  logging:
    driver: json-file

web:
  <<: *defaults        # merge key: pulls in everything from &defaults
  image: nginx:1.25

worker:
  <<: *defaults
  image: worker:2.0
  restart: on-failure   # overrides the merged value
```

`&defaults` creates an **anchor** (a named reference to that node), `*defaults` is an **alias** that reuses it, and `<<:` is the **merge key** — it splices the anchored mapping's keys into the current mapping, with locally defined keys taking precedence. The gotcha: `<<:` only merges **top-level** keys — it does not deep-merge nested mappings. If `&defaults` has `logging: {driver: json-file}` and `worker` also defines `logging: {options: {max-size: "10m"}}`, worker's `logging` **replaces** the anchor's entire `logging` value; `driver` is gone, not merged with `options`. A YAML file's *effective* content can differ substantially from what's visible top-to-bottom without tracing every anchor reference.

### The "Norway problem" and implicit type coercion

This is the most-tested YAML gotcha in interviews and a genuine, recurring source of production bugs. YAML 1.1 (what PyYAML implements by default, regardless of what version the file "targets") resolves **unquoted scalars** to typed values using an implicit-typing table that's far more aggressive than most people expect:

```python
import yaml

data = yaml.safe_load("""
country_code: NO
feature_enabled: yes
retry: off
version: 1.10
file_mode: 0755
port: 08080
release: 2024-01-15
empty_value:
""")

for k, v in data.items():
    print(f"{k:16} -> {v!r:26} ({type(v).__name__})")
```

Verified output:

```
country_code    -> False                      (bool)
feature_enabled -> True                       (bool)
retry           -> False                      (bool)
version         -> 1.1                        (float)
file_mode       -> 493                        (int)
port            -> '08080'                    (str)
release         -> datetime.date(2024, 1, 15) (date)
empty_value     -> None                       (NoneType)
```

Walk through what happened:
- **`country_code: NO`** — the ISO code for Norway parses as the boolean `False`. This is *the* "Norway problem." YAML 1.1's boolean set includes `y/n/yes/no/on/off/true/false` in any case, so `NO`, `No`, `no` are all booleans, not the string `"NO"`.
- **`feature_enabled: yes` / `retry: off`** — same mechanism: `yes`/`off` become booleans, not the literal words a downstream system might expect.
- **`version: 1.10`** — an unquoted number-looking scalar parses as a float, and floats normalize trailing zeros away: `1.10` becomes `1.1`, indistinguishable from what `1.1` would have parsed to. Disastrous for anything resembling a semantic version stored unquoted.
- **`file_mode: 0755`** — a leading zero followed only by valid octal digits (0–7) is parsed as **octal**: `0755` silently becomes decimal `493`. Genuinely dangerous for Unix file modes or zero-padded values stored unquoted.
- **`port: 08080`** — note this one does *not* become an octal int, because `8` isn't a valid octal digit — PyYAML's implicit-int regex doesn't match a leading-zero string containing 8 or 9, so it falls through and stays the string `"08080"`. This inconsistency (some leading-zero numbers become octal ints, others silently stay strings, depending on which digits happen to appear) is exactly why you can't eyeball YAML numeric-looking values and know their type — you have to load and print the type to be sure.
- **`release: 2024-01-15`** — an unquoted ISO-8601-looking string becomes a `datetime.date` object via YAML's implicit timestamp type, not a string. Code that calls `.split("-")` expecting a string will crash with an `AttributeError`.
- **`empty_value:`** (nothing after the colon) — parses to `None`, same as an explicit `null`, `~`, or `Null`.

**The fix is always the same: quote anything that isn't semantically a number, boolean, or timestamp.** `country_code: "NO"`, `version: "1.10"`, `file_mode: "0755"`. YAML 1.2 (the current spec) tightened the core schema significantly — only `true`/`false` are booleans (no `yes`/`no`/`on`/`off`), and octal requires an explicit `0o` prefix — but **PyYAML implements YAML 1.1 rules unconditionally**, no matter what you name the file or what version header you add. `ruamel.yaml` can be pinned to 1.2 core-schema behavior (shown below), which removes the `yes`/`no`/`on`/`off` coercion — a useful mitigation, but don't rely on switching libraries as your primary defense. Quote your scalars, and validate structurally with a schema (below) so a coercion bug gets caught at load time, not at 2am in prod.

## PyYAML vs ruamel.yaml

**PyYAML** is the ubiquitous, load-and-dump library — fast, simple, but it treats a YAML file as a one-way trip into plain Python types (`dict`, `list`, `str`, `int`, `float`, `bool`, `None`). Dump it back out and all comments, key-ordering quirks, blank lines, and quote styles are gone.

```python
import yaml

with open("config.yaml") as f:
    config = yaml.safe_load(f)

config["replicas"] = 5
print(yaml.safe_dump(config, sort_keys=False, default_flow_style=False))
```

**Never use `yaml.unsafe_load()` or `Loader=yaml.UnsafeLoader`/`Loader=yaml.Loader` on any file you don't fully control.** Those loaders honor tags like `!!python/object/apply:os.system [...]` that construct and **call** arbitrary Python callables during parsing — verified: a crafted YAML document with that tag runs the shell command the instant `yaml.load(payload, Loader=yaml.UnsafeLoader)` executes. This is a deserialization vulnerability in the same class as unpickling untrusted data (CWE-502): a pulled-in Helm chart, a user-submitted config in a multi-tenant tool, or any YAML from outside your own commits becomes a code-execution vector. Modern PyYAML (5.1+) requires an explicit `Loader=` argument — bare `yaml.load(f)` now raises `TypeError: load() missing 1 required positional argument: 'Loader'` rather than silently picking an unsafe default — but older/pinned PyYAML versions defaulted `yaml.load()` to the unsafe loader, which is exactly why the CVEs (and the "never use bare `yaml.load()`" folklore) exist. `FullLoader` (PyYAML's post-5.1 default target for `yaml.full_load()`) blocks the `!!python/object/apply` code-execution path specifically — confirmed: the same payload raises `ConstructorError` under `Loader=yaml.FullLoader` instead of executing — but it is still not the library's recommended choice for untrusted input. `yaml.safe_load()` restricts construction to plain built-in types only and is what you should reach for by default, for trusted and untrusted YAML alike, because a config file legitimately never needs to construct any Python object graph, safe or not.

**ruamel.yaml** targets the case PyYAML doesn't cover: you need to **programmatically edit** a YAML file that a human also maintains, without your edit blowing away their comments and formatting.

```python
from ruamel.yaml import YAML

yaml = YAML()                      # default typ='rt' = round-trip
yaml.preserve_quotes = True
yaml.indent(mapping=2, sequence=4, offset=2)

with open("app-config.yaml") as f:
    data = yaml.load(f)

data["replicas"] = 5                # change exactly one value

with open("app-config.yaml", "w") as f:
    yaml.dump(data, f)
```

Given an input file:

```yaml
# Staging config for payments-api
service_name: payments-api
replicas: 2       # keep low in staging to save cost
image: registry.example.com/payments-api:1.4.2
```

ruamel.yaml's round-trip mode reproduces the file with only `replicas` changed — the header comment, the inline comment, and the key order all survive. The same edit through PyYAML's `safe_load`/`safe_dump` produces a structurally-equivalent but comment-free, potentially re-ordered file — indistinguishable in *content*, but a large, noisy diff and a lost comment in code review. Use ruamel.yaml whenever a script edits YAML a human will read/diff afterward (bumping a version, toggling a flag in CI); use plain PyYAML when you only ever read config to consume it, or generate brand-new YAML with no existing formatting to preserve.

ruamel.yaml also lets you pin the YAML **spec version** to get 1.2 core-schema type resolution instead of 1.1's:

```python
from ruamel.yaml import YAML

y = YAML(typ="safe")
y.version = (1, 2)
data = y.load(open("gotchas.yaml"))
# country_code -> 'NO' (str, not bool)  — 1.2 core schema only recognizes true/false as booleans
# retry        -> 'off' (str, not bool)
# file_mode    -> 755 (decimal int, not octal 493) — 1.2 dropped implicit octal-via-leading-zero
```

It also exposes a `typ="unsafe"` mode analogous to PyYAML's unsafe loader — same warning applies: never point it at untrusted input.

## JSON Schema validation with `jsonschema`

Loading YAML with `safe_load` only guarantees you get *some* combination of dicts, lists, strings, numbers, booleans, and `None` — it guarantees nothing about **shape**: whether required keys are present, whether `replicas` is actually an integer and not the string `"three"`, whether `image` matches a valid `repo:tag` pattern. Feeding unvalidated config straight into business logic means a typo surfaces as a `KeyError`/`TypeError` deep inside a deployment script — or worse, as a Kubernetes API rejection after a rollout has already started — instead of as an immediate, clear "config is invalid" error at load time.

**JSON Schema** describes the shape of JSON-like data (works fine against parsed YAML too, since the two data models overlap almost completely once parsed). A schema is itself a JSON/YAML document:

```python
schema = {
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "type": "object",
    "required": ["service_name", "replicas", "image"],
    "additionalProperties": False,
    "properties": {
        "service_name": {
            "type": "string",
            "pattern": "^[a-z][a-z0-9-]*$",
            "maxLength": 63
        },
        "replicas": {"type": "integer", "minimum": 1, "maximum": 50},
        "image": {
            "type": "string",
            "pattern": r"^[\w./-]+:[\w.-]+$"
        },
        "env": {
            "type": "object",
            "additionalProperties": {"type": "string"}
        },
        "resources": {
            "type": "object",
            "required": ["cpu", "memory"],
            "properties": {
                "cpu": {"type": "string"},
                "memory": {"type": "string"}
            }
        }
    }
}
```

Key building blocks: `type` (string/integer/number/boolean/object/array/null), `required` (mandatory keys), `properties` (per-key sub-schemas), `additionalProperties: False` (reject unknown keys — catches typos like `replias` instead of `replicas` immediately instead of silently ignoring them), `pattern` (regex constraint on strings), `enum` (closed set of allowed values), `minimum`/`maximum`, and nested `object`/`properties` for structure. `additionalProperties` **defaults to `True`** — if you want typo protection you must set it explicitly.

Validating and reporting *every* problem at once, rather than stopping at the first, is what you want for a config file a human will fix:

```python
import yaml
from jsonschema import Draft202012Validator

config = yaml.safe_load(open("app-config.yaml"))

validator = Draft202012Validator(schema)
errors = sorted(validator.iter_errors(config), key=lambda e: list(e.path))

if errors:
    for error in errors:
        path = ".".join(str(p) for p in error.path) or "<root>"
        print(f"  {path}: {error.message}")
    raise SystemExit(f"{len(errors)} config error(s) — fix app-config.yaml before deploying")
```

For a one-shot check where the first error is enough, `jsonschema.validate(instance=config, schema=schema)` raises `jsonschema.ValidationError` on the first failure — simpler, but worse UX for a human iterating on a config file. The discipline this buys you: **fail fast, fail at the boundary.** Validate config the moment it's loaded, before it's passed one function deeper — the difference between a CI job failing in 2 seconds with "`replicas` must be >= 1" and a production rollout failing 10 minutes in with a cryptic Kubernetes API error because `replicas: "3"` (a string) got through unvalidated.

## Points to Remember

- YAML is indentation-sensitive and tab-forbidden; mixed indent widths at the same level are a parse error, not a silent success.
- `safe_load()` on a `---`-separated multi-document file raises `ComposerError` rather than silently returning only the first document — use `safe_load_all()` (returns a generator) whenever a file might contain more than one document.
- Anchors (`&name`) + aliases (`*name`) + merge keys (`<<:`) let you reuse a YAML node — but merge keys only merge **top-level** keys, they don't deep-merge nested mappings.
- The "Norway problem": unquoted `NO`/`no`/`yes`/`on`/`off` parse as booleans under YAML 1.1 (PyYAML's default, unconditionally). Always quote strings that could be misread as bool/int/float/timestamp.
- Leading-zero unquoted numbers made only of octal digits (`0755`) are parsed as octal; the same pattern with an 8 or 9 in it (`08080`) stays a string — this inconsistency is precisely why you should never guess a YAML scalar's type by eye.
- `yaml.safe_load()` is the default choice always. `yaml.unsafe_load()` / `Loader=yaml.UnsafeLoader` / `Loader=yaml.Loader` can construct **and call** arbitrary Python callables from the file (a CWE-502-class deserialization risk, verified, not theoretical); modern PyYAML at least forces you to pass `Loader=` explicitly rather than silently defaulting to one of these.
- PyYAML: fast, simple, but round-tripping loses comments/formatting/order. ruamel.yaml: preserves comments/formatting on round-trip, and can be pinned to YAML 1.2 core-schema typing rules.
- `jsonschema` validates *shape*, not just "did it parse" — required keys, types, patterns, ranges, and disallowed unknown keys (`additionalProperties: False`, which defaults to `True`) catch typos and malformed config before they reach application logic.

## Common Mistakes

- Writing `enabled: yes` or `country: NO` unquoted and being surprised the value is a Python `bool`, not the string intended — quote any value that could be misread as boolean, numeric, or a timestamp.
- Reaching for `yaml.unsafe_load()`/`Loader=yaml.UnsafeLoader`/`Loader=yaml.Loader` (sometimes copy-pasted from an old Stack Overflow answer predating PyYAML 5.1) on config that could ever originate outside your own commits — a real code-execution vector, not a hypothetical one.
- Calling `yaml.safe_load()` on a real-world Kubernetes manifest or Helm template output that happens to bundle multiple `---`-separated resources, and hitting `ComposerError` in what looked like a simple parsing step — `safe_load_all()` was needed.
- Assuming `additionalProperties` defaults to `False` in JSON Schema — it defaults to `True` (unknown keys silently allowed). Without setting it explicitly, typo'd keys pass validation.
- Treating schema validation as optional and skipping it in a rushed pipeline change — the entire point is catching a bad config in CI, not mid-rollout in production.
- Round-tripping a human-maintained YAML file through plain PyYAML `safe_load`/`safe_dump` in an automation script, then wondering why every automated PR touching that file produces a comment-stripped, reordered diff — that's a ruamel.yaml use case, not a PyYAML one.
- Assuming a leading zero always means "octal" (or always means "string") — it depends on the exact digits present, which is itself the argument for quoting rather than relying on parser behavior.
