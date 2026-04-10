# Quickstart: F4 — RCSA Assets

**Feature**: F4 — Templates, Sample Data, and Control Library
**Prerequisite**: Repository cloned, Python 3.11+ available

---

## What F4 Is

F4 is a static asset package — 6 files, zero runtime code. It provides:
- **3 JSON sample data files** — inputs that F6 scripts consume
- **2 Markdown templates** — output structures that F6 populates
- **1 citation format spec** — syntax rules for inline evidence citations

---

## Directory Layout

```text
skills/rcsa/assets/
├── templates/
│   ├── rcsa_control_narratives_template.md
│   └── validation_report_template.md
├── sample-data/
│   ├── sample_control_library.json
│   ├── sample_artifact_index.json
│   └── sample_test_catalog.json
└── citation_format.md
```

---

## Validation

### Quick Check (Python stdlib — no dependencies)

Verify all JSON files parse without errors:

```bash
python3 -c "
import json, pathlib
base = pathlib.Path('skills/rcsa/assets/sample-data')
for f in base.glob('*.json'):
    json.load(open(f))
    print(f'✅ {f.name}')
"
```

### Schema Validation (requires `jsonschema` package)

Validate sample data against F4's contracts:

```bash
pip install jsonschema

python3 -c "
import json, jsonschema, pathlib

pairs = [
    ('control_library.schema.json', 'sample_control_library.json'),
    ('artifact_index.schema.json', 'sample_artifact_index.json'),
    ('test_catalog.schema.json', 'sample_test_catalog.json'),
]

contracts = pathlib.Path('specs/f4-rcsa-assets/contracts')
data = pathlib.Path('skills/rcsa/assets/sample-data')

for schema_file, data_file in pairs:
    schema = json.load(open(contracts / schema_file))
    instance = json.load(open(data / data_file))
    jsonschema.validate(instance, schema)
    print(f'✅ {data_file} matches {schema_file}')
"
```

### Manual Checks

| Check | How |
|---|---|
| UTF-8 encoding | `file --mime-encoding skills/rcsa/assets/**/*` — all files should report `utf-8` |
| No sensitive data | `grep -rE '(password\|secret\|api.key\|@.*\.com\|\d{3}-\d{2}-\d{4})' skills/rcsa/assets/` — should return nothing |
| Relative paths only | Inspect JSON files — no `file_path` value should start with `/` |
| Template placeholders | Open both templates — verify all placeholders use `{{field_name}}` syntax consistently |

---

## Evidence Coverage Design

F4's sample data is intentionally designed with mixed coverage:

| Control | Evidence Provided | Gaps | Expected Confidence |
|---|---|---|---|
| AC (Access Control) | 3/3 types | None | HIGH |
| CM (Change Management) | 3/3 types | None | HIGH |
| DQ (Data Quality) | 1/2 types | `data_transform_test` missing | MEDIUM |
| IH (Incident Handling) | 1/3 types | `incident_response_plan`, `postmortem_record` missing | LOW |

This exercises the full range of confidence tiers and gap detection in the pipeline.

---

## How F4 Connects to F5 and F6

```text
F5 (SKILL.md)          F4 (Assets)              F6 (Scripts)
─────────────          ──────────               ───────────
References paths  ──→  templates/               ←── Output assembler populates
References format ──→  citation_format.md       ←── Citation validator checks
                       sample-data/             ←── Input validator + evidence mapper read
```

- **F5** references F4's file paths and citation format in its LLM instructions
- **F6** reads F4's sample data as inputs and populates F4's templates as outputs
- **F4 owns the contracts** — F5 and F6 conform to F4's schemas, not the other way around

---

## Re-validation Gates

After downstream features are built, re-validate F4:

| Gate | When | What to Check |
|---|---|---|
| G2.1 | After F5 is built | F5's LLM output instructions match F4's template YAML fields |
| G2.2 | After F6 plan is approved | F6's input expectations match F4's JSON schemas. **Blocking** — F6 implementation cannot start until this passes. |
