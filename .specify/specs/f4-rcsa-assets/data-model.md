# Data Model: F4 — RCSA Assets

**Feature**: F4 — Templates, Sample Data, and Control Library
**Date**: 2026-04-10
**Formal schemas**: See `contracts/` for JSON Schema definitions

---

## Sample Data Files

### `sample_control_library.json`

Top-level structure:

| Field | Type | Required | Description |
|---|---|---|---|
| `_meta` | object | Yes | Metadata block indicating scale and handoff notes |
| `_meta.scale` | string | Yes | `"demo"` — indicates demo-scale data |
| `_meta.note` | string | Yes | Handoff instruction for Synchrony |
| `controls` | array | Yes | Array of control objects (exactly 4 for demo) |

Control object:

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | Yes | Short identifier: `AC`, `CM`, `DQ`, `IH` |
| `name` | string | Yes | Human-readable control name |
| `objective` | string | Yes | One-sentence description of what the control ensures |
| `evidence_types` | array | Yes | Array of evidence type objects (≥2 per control) |

Evidence type object:

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | Yes | Snake_case identifier (e.g., `auth_config`) |
| `label` | string | Yes | Human-readable label (e.g., "Authentication Configuration") |
| `description` | string | Yes | What this evidence type represents |

---

### `sample_artifact_index.json`

Top-level structure:

| Field | Type | Required | Description |
|---|---|---|---|
| `_meta` | object | Yes | Metadata block (same structure as control library) |
| `artifacts` | array | Yes | Array of artifact objects (10–20 for demo) |

Artifact object:

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | Yes | Unique identifier (e.g., `ART-001`) |
| `file_path` | string | Yes | Relative path to the artifact file |
| `file_type` | string | Yes | File extension/type (e.g., `yaml`, `json`, `python`) |
| `description` | string | Yes | What this artifact contains |
| `lines` | string | No | Line range if referencing a subset (e.g., `"1-42"`) |

---

### `sample_test_catalog.json`

Top-level structure:

| Field | Type | Required | Description |
|---|---|---|---|
| `_meta` | object | Yes | Metadata block (same structure as control library) |
| `tests` | array | Yes | Array of test objects (5–10 for demo) |

Test object:

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | Yes | Unique identifier (e.g., `TST-001`) |
| `file_path` | string | Yes | Relative path to the test file |
| `file_type` | string | Yes | File extension/type (e.g., `python`, `shell`) |
| `description` | string | Yes | What this test validates |
| `controls_relevant` | array | Yes | Array of control IDs this test relates to (e.g., `["AC"]`) |

---

## Template Definitions

### `rcsa_control_narratives_template.md`

**YAML Front Matter Fields**:

| Field | Type | Description |
|---|---|---|
| `title` | string | `"RCSA Control Narratives"` |
| `generated` | string | ISO 8601 timestamp |
| `skill_version` | string | Skill version identifier |
| `total_controls` | integer | Number of controls evaluated |
| `overall_confidence` | string | Aggregate confidence tier (`HIGH`, `MEDIUM`, `LOW`) |

**Document Sections** (in order):

1. **Summary Table** — columns: Control | Confidence Tier | Evidence Found | Gaps Identified

2. **Per-Control Sections** (repeated for each control):
   - `## {control_name} ({control_id})` — section heading
   - Narrative paragraph with inline `[file_path — description]` citations
   - Gap callout block (if applicable): `> ⚠️ GAP: {evidence_type} — no matching evidence found`
   - Evidence list: bullet list of matched evidence with citation tags

**Placeholder Syntax**: `{{field_name}}` — all placeholders use double curly braces for F6 find-and-replace.

---

### `validation_report_template.md`

**YAML Front Matter Fields**:

| Field | Type | Description |
|---|---|---|
| `title` | string | `"Validation Report"` |
| `generated` | string | ISO 8601 timestamp |
| `total_citations` | integer | Total citation tags in the narratives |
| `valid_citations` | integer | Citations that resolve to a real file path |
| `invalid_citations` | integer | Citations that fail to resolve |
| `resolution_rate` | string | Percentage of valid citations (e.g., `"85%"`) |

**Document Sections** (in order):

1. **Citation Index Table** — columns: Citation | File Path | Lines | Status (`✅ Valid` / `❌ Invalid`)

2. **Evidence Coverage Table** — columns: Control | Expected Types | Matched | Unmatched | Coverage %

3. **Flagged for Human Review** — bullet list of specific gaps, unresolved citations, or anomalies requiring follow-up

**Placeholder Syntax**: `{{field_name}}` — same convention as narratives template.

---

### `citation_format.md`

This file is a specification, not a template. No placeholders.

| Element | Definition |
|---|---|
| **Tag syntax** | `[file_path — description]` where `file_path` is the relative path to the evidence file and `description` is a brief human-readable label |
| **Separator** | ` — ` (space, em dash, space) divides the machine-readable path from the human-readable description |
| **Multi-file evidence** | Use separate citations per file — no sub-index syntax needed |
| **Resolution** | Each citation is self-resolving — the file path is embedded directly in the tag |
| **Examples** | Minimum 2 controls with worked examples showing citation usage in narrative text |

---

## Cross-References

- Formal JSON Schema definitions: `contracts/`
- Coverage design (which evidence types are present/missing): `plan.md` → Design Decisions Log
- Acceptance scenarios validating these schemas: `plan.md` → AS-01 through AS-05, AS-12
