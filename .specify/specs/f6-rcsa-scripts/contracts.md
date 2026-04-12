# Contract: F6 ↔ F4 (Scripts ↔ Assets)

**Feature**: F6 — RCSA Scripts | **Date**: 2026-04-12 | **Status**: Draft
**Parties**: F6 (consumer) ← F4 (provider)

---

## Purpose

This contract defines what F6 expects from F4. F4 provides the input data files and output templates. F6 reads them, validates them, and uses them to produce the final output. If F4 changes any of these structures, F6 must be updated.

---

## 1. Input Files

F4 provides 3 JSON files. F6 reads all 3.

### 1.1 Control Library

| Property | Value |
|---|---|
| **File** | `sample_control_library.json` |
| **Location** | `.cursor/skills/rcsa/data/` |
| **Format** | JSON, UTF-8 |
| **Root key** | `controls` (array, required, non-empty) |

**Required fields per control:**

| Field | Type | Constraint | Example |
|---|---|---|---|
| `id` | string | `^[A-Z]{2,4}$` | `"AC"` |
| `name` | string | Non-empty | `"Access Control"` |
| `objective` | string | Non-empty | `"Ensure system access is restricted..."` |
| `evidence_types` | array | Non-empty, at least 1 entry | See below |

**Required fields per evidence type:**

| Field | Type | Constraint | Example |
|---|---|---|---|
| `id` | string | Non-empty, unique within control | `"auth_config"` |
| `label` | string | Non-empty | `"Authentication Configuration"` |
| `description` | string | Non-empty | `"Configuration files defining..."` |

**Optional fields (ignored by F6):**

| Field | Notes |
|---|---|
| `_meta` | Metadata object. F6 skips it. |

**Demo data contract:**
- Exactly 4 controls: `AC`, `CM`, `DQ`, `IH`
- 11 total evidence types across all controls
- If a 5th control is added, F6 will process it — no code change needed
- If a control ID violates `^[A-Z]{2,4}$`, `validate_input.py` rejects the file

---

### 1.2 Artifact Index

| Property | Value |
|---|---|
| **File** | `sample_artifact_index.json` |
| **Location** | `.cursor/skills/rcsa/data/` |
| **Format** | JSON, UTF-8 |
| **Root key** | `artifacts` (array, required, non-empty) |

**Required fields per artifact:**

| Field | Type | Constraint | Example |
|---|---|---|---|
| `id` | string | Format: `ART-NNN` | `"ART-001"` |
| `file_path` | string | Non-empty, unique across all artifacts | `"auth/oauth_config.yaml"` |
| `file_type` | string | Non-empty | `"yaml"` |
| `description` | string | Non-empty | `"OAuth2 provider configuration..."` |

**Demo data contract:**
- 15 artifacts
- No duplicate `id` or `file_path` values
- `file_path` values are used in citations — they must match exactly what the LLM cites

---

### 1.3 Test Catalog

| Property | Value |
|---|---|
| **File** | `sample_test_catalog.json` |
| **Location** | `.cursor/skills/rcsa/data/` |
| **Format** | JSON, UTF-8 |
| **Root key** | `tests` (array, required, non-empty) |

**Required fields per test:**

| Field | Type | Constraint | Example |
|---|---|---|---|
| `id` | string | Format: `TST-NNN` | `"TST-001"` |
| `file_path` | string | Non-empty, unique across all tests | `"tests/auth/test_rbac_permissions.py"` |
| `file_type` | string | Non-empty | `"python"` |
| `description` | string | Non-empty | `"Verifies that RBAC role assignments..."` |
| `controls_relevant` | array[string] | Non-empty, each value must match a `controls[].id` | `["AC"]` |

**Demo data contract:**
- 8 tests
- Every value in `controls_relevant` must exist in the control library
- A test can map to multiple controls (e.g., `["AC", "CM"]`)

---

## 2. Citation Format

| Property | Value |
|---|---|
| **Defined in** | F4 `citation_format.md` |
| **Format** | `[file_path — description]` |
| **Example** | `[auth/oauth_config.yaml — OAuth2 provider configuration]` |
| **Regex** | `\[([^\]]+?)\s—\s([^\]]+?)\]` |

**Contract terms:**
- `file_path` (group 1) MUST match an entry in `artifacts[].file_path` or `tests[].file_path`
- `description` (group 2) is logged but not validated against the source data
- The em dash ` — ` (space, em dash, space) is the delimiter — not a hyphen
- F5 instructs the LLM to use this format
- F6 parses and validates citations against this format

**If F4 changes the citation format:**
- F5's SKILL.md must be updated (LLM instructions)
- F6's `validate_citations.py` regex must be updated
- All three features must be re-aligned before deployment

---

## 3. Output Templates (if applicable)

| Property | Value |
|---|---|
| **Status** | To be confirmed |
| **Expected location** | `.cursor/skills/rcsa/templates/` |

If F4 provides Markdown templates with `{{placeholder}}` syntax, `write_output.py` will populate them. If F4 does not provide templates, `write_output.py` will generate the output structure directly based on the schemas defined in `data-model.md`.

**Placeholder syntax (if templates are used):**

| Placeholder | Replaced With |
|---|---|
| `{{title}}` | Document title from YAML front matter |
| `{{generated}}` | ISO 8601 timestamp |
| `{{controls_assessed}}` | Number of controls |
| `{{summary_table}}` | Markdown table of all controls |
| `{{control_AC_narrative}}` | Narrative text for AC |
| `{{control_AC_confidence}}` | Confidence tier for AC |
| `{{control_[ID]_narrative}}` | Narrative text for any control |
| `{{control_[ID]_confidence}}` | Confidence tier for any control |

---

## 4. Breaking Changes

Any of the following changes in F4 **break F6** and require coordinated updates:

| Change | Impact on F6 |
|---|---|
| Rename `controls` root key | `validate_input.py` and `build_registry.py` fail |
| Change control ID format | `validate_input.py` regex fails |
| Remove `evidence_types` from controls | `map_evidence.py` cannot map artifacts to controls |
| Change `file_path` field name in artifacts | `build_registry.py` file path index breaks |
| Change citation format | `validate_citations.py` regex fails |
| Add required fields | F6 won't validate them unless updated |

---

*This contract is owned jointly by F4 and F6. Changes require review by both sides.*
