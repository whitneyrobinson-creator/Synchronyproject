# F2 — Scripts: Data Dictionary Generation

## Feature Specification

---

## 1. Feature Identity

| Field | Value |
|---|---|
| **Feature Name** | F2 — Scripts: Data Dictionary Generation |
| **Feature Branch** | `f2-scripts-data-dictionary` |
| **Created** | 2026-04-03 |
| **Status** | Draft |
| **Owner** | Whitney Robinson (PM), Sheila Green, Molly Lowell |
| **Demo Day Deadline** | May 7, 2026 |
| **Input** | User provides a JSON schema file describing a database table. F2 scripts parse, validate, extract, and assemble — the LLM (F1) handles reasoning in between. |

---

## 2. User Scenarios

> **Note:** All script output is **deterministic** — given the same input, scripts always produce the same output. The scripts handle everything that does NOT require an LLM: parsing, validating, extracting, citing, timestamping, assembling, and reporting.

> **Pipeline Flow Reference:**
>
> ```
> [Pre-LLM]  US-1 → US-2 → [F1/LLM generates descriptions] → [Post-LLM] US-3 → US-4 → US-5
>                                                                          ↕
>                                                               US-6 (fallback if LLM fails)
> US-7 (P3, future scope — runs pre-LLM if glossary provided)
> ```

---

### US-1: Parse and Validate a Schema File (P1)

**Pipeline Position:** Pre-LLM — Step 1.

**As a** documentation team member,
**I want** the system to accept my JSON schema file, validate that it's well-formed and structurally correct, and tell me about all problems at once if something is wrong,
**So that** I can fix any issues in one pass before the pipeline runs.

**Priority:** P1 — Required for demo day.

**Script:** `validate_input.py`

**Acceptance Scenarios:**

| # | Given | When | Then |
|---|---|---|---|
| 1.1 | A valid JSON schema file with `table_name`, `source_file`, and a non-empty `fields` array where every field has a non-empty `field_name` | The user provides the file to the skill | `validate_input.py` passes with no errors. Writes `validated_schema.json`. Pipeline proceeds. |
| 1.2 | A JSON file missing `table_name` and containing two fields without `field_name` | The user provides the file to the skill | `validate_input.py` collects all errors and returns them in a single error report. Pipeline does not proceed. |
| 1.3 | A file that is not valid JSON (e.g., trailing commas, missing brackets) | The user provides the file to the skill | `validate_input.py` catches the parse error and returns a user-friendly error with line/column number and hint. Pipeline does not proceed. |
| 1.4 | A YAML or DDL file instead of JSON | The user provides the file to the skill | `validate_input.py` returns a clear error: "Only JSON schema files are supported in this version. Please convert to JSON." |
| 1.5 | A valid JSON file where `fields` is an empty array | The user provides the file to the skill | `validate_input.py` returns error: "Schema contains no extractable fields." Pipeline does not proceed. |
| 1.6 | A valid JSON file where a `field_name` is empty, whitespace-only, or exceeds 255 characters | The user provides the file to the skill | `validate_input.py` reports the specific field position and error. Pipeline does not proceed. |

---

### US-2: Extract Field Metadata from a Valid Schema (P1)

**Pipeline Position:** Pre-LLM — Step 2.

**As a** documentation team member,
**I want** the system to extract all field-level metadata from my schema file and construct a traceable source path for each field,
**So that** the LLM has everything it needs to write accurate descriptions, and every field is traceable back to its origin.

**Priority:** P1 — Required for demo day.

**Script:** `extract_fields.py`

**Acceptance Scenarios:**

| # | Given | When | Then |
|---|---|---|---|
| 2.1 | A valid schema with 25 fields with complete metadata | `extract_fields.py` processes `validated_schema.json` | Output `extracted_fields.json` contains 25 objects, each with exactly 8 keys. `source_path` is constructed as `{source_file} → table: {table_name} → field: {field_name}`. |
| 2.2 | A valid schema where some fields are missing `type`, `enums`, or `schema_comments` | `extract_fields.py` processes `validated_schema.json` | Missing optional fields are filled with deterministic defaults: `type` → `"UNKNOWN"`, `nullable` → `true`, `constraints` → `[]`, `enums` → `[]`, `schema_comments` → `null`. Pipeline continues. |
| 2.3 | A valid schema with duplicate `field_name` values | `extract_fields.py` processes `validated_schema.json` | Both entries are included. Duplicates logged to `extraction_warnings.json`. |
| 2.4 | A field with leading/trailing whitespace or newlines in `field_name` | `extract_fields.py` processes the field | Whitespace and newlines stripped. Correction logged as `field_name_whitespace_stripped` or `field_name_newline_stripped`. |
| 2.5 | A field where `nullable` is provided as a string `"true"` or `"false"` | `extract_fields.py` processes the field | Coerced to boolean. Correction logged as `nullable_type_coerced`. |

---

### US-3: Validate and Merge LLM Output with Citations (P1)

**Pipeline Position:** Post-LLM — Step 1.

**As a** documentation team member,
**I want** the system to validate that the LLM returned output for every field, merge the LLM's descriptions and scores with my original metadata, and clearly flag any fields the LLM missed,
**So that** I can trust the data dictionary is complete and every field is traceable.

**Priority:** P1 — Required for demo day.

**Script:** `attach_citations.py`

**Acceptance Scenarios:**

| # | Given | When | Then |
|---|---|---|---|
| 3.1 | LLM returns a valid JSON array with 25 objects, all `field_name` values matching | `attach_citations.py` runs | `merged_fields.json` contains 25 objects with 14 keys each: all metadata + `description`, `confidence`, `evidence_refs`, `clarification_flag`, `merge_status`, `corrections`. |
| 3.2 | LLM returns 23 objects instead of 25 | `attach_citations.py` runs | Count mismatch logged. 23 matched fields merged normally. 2 unmatched fields get placeholder values. `merge_status` = `"placeholder"` for missing fields. |
| 3.3 | LLM returns a `field_name` that doesn't match any input field | `attach_citations.py` runs | Orphaned LLM output ignored. Unmatched input field gets placeholder values. |
| 3.4 | LLM output has structural failures (wrong confidence value, missing keys, non-boolean flag) | `attach_citations.py` runs | Per-field structural failures → only affected field gets placeholders. Root-level structural failure → all fields get placeholders. All failures logged to Warnings. |
| 3.5 | LLM returns `confidence` = `"Low"` but `clarification_flag` = `false` | `attach_citations.py` runs | Flag overridden to `true`. Correction logged as `clarification_flag_override`. |
| 3.6 | LLM output fails validation — F2 retries | `attach_citations.py` runs up to 3 total attempts | If all 3 attempts fail, all fields get placeholders. Both deliverables still produced. |

---

### US-4: Timestamp and Assemble the Data Dictionary (P1)

**Pipeline Position:** Post-LLM — Steps 2 and 3.

**As a** documentation team member,
**I want** the system to add a `last_verified` timestamp to every field and assemble all data into a formatted data dictionary,
**So that** I receive one audit-ready deliverable I can review and share.

**Priority:** P1 — Required for demo day.

**Scripts:** `add_timestamps.py` → `assemble_output.py`

**Acceptance Scenarios:**

| # | Given | When | Then |
|---|---|---|---|
| 4.1 | `merged_fields.json` contains 25 fully merged field objects | `add_timestamps.py` runs | `timestamped_fields.json` contains all 25 objects with 15 keys each. All fields share one ISO 8601 UTC timestamp per run. |
| 4.2 | `timestamped_fields.json` is ready and `data_dictionary_template.md` exists in `assets/` | `assemble_output.py` runs | `data_dictionary.md` is produced. Header placeholders replaced using `.replace()` in fixed order. Special characters in table cells escaped via `sanitize_for_markdown()`. Enum values separated by semicolons. Evidence refs rendered as numbered list. |
| 4.3 | A field has `clarification_flag` = `true` | `assemble_output.py` runs | `[NEEDS CLARIFICATION]` prepended to the field's description in the output. |
| 4.4 | Template file is missing | `assemble_output.py` runs | Script stops immediately with clear error message. This is a setup error, not graceful degradation. |
| 4.5 | Template file contains Git merge conflict markers | `assemble_output.py` runs | Script stops with error before any parsing or substitution. |
| 4.6 | The skill is run twice on the same input | Both runs complete | Outputs identical except `last_verified` timestamp. |

---

### US-5: Generate the QA Report (P1)

**Pipeline Position:** Post-LLM — Step 4 (final).

**As a** documentation team member,
**I want** the system to produce a QA report showing coverage stats, confidence distribution, flagged items, and any warnings,
**So that** I know exactly what needs my attention and can verify completeness at a glance.

**Priority:** P1 — Required for demo day.

**Script:** `generate_qa_report.py`

**Acceptance Scenarios:**

| # | Given | When | Then |
|---|---|---|---|
| 5.1 | A complete data dictionary with 25 fields, all with descriptions | `generate_qa_report.py` runs | `qa_report.md` shows all 6 sections with correct counts. Confidence distribution sums to 25. Completeness = 100%. |
| 5.2 | A partial data dictionary where 2 fields have placeholders | `generate_qa_report.py` runs | Completeness = 92%. 2 missing fields listed. `merge_status` = `"placeholder"` used to calculate coverage. |
| 5.3 | LLM was offline — all fields are placeholders | `generate_qa_report.py` runs | Completeness = 0%. Processing Notes shows `Partial` status. Clear note that LLM content is missing. |
| 5.4 | `extraction_warnings.json` exists | `generate_qa_report.py` runs | Warnings section includes duplicate field name entries. |
| 5.5 | Confidence distribution sum does not equal total field count | `generate_qa_report.py` runs | Pipeline integrity warning logged. Pipeline continues — no crash. |
| 5.6 | Template file is missing or contains conflict markers | `generate_qa_report.py` runs | Script stops with clear error before any processing. |

---

### US-6: Graceful Degradation — LLM Offline or Failed (P1)

**Pipeline Position:** Fallback path.

**As a** documentation team member,
**I want** the system to still produce a useful partial data dictionary and QA report even when the LLM is unavailable,
**So that** I have all the raw metadata organized and ready.

**Priority:** P1 — Required for demo day. Constitution Principle 2 (Graceful Degradation).

**Scripts:** All scripts still run. `attach_citations.py` detects the failure and fills placeholders.

**Acceptance Scenarios:**

| # | Given | When | Then |
|---|---|---|---|
| 6.1 | `llm_output.json` does not exist | `attach_citations.py` runs | All fields get placeholder values. `merge_status` = `"placeholder"`. Pipeline continues. |
| 6.2 | `llm_output.json` exists but is not parseable JSON | `attach_citations.py` runs | Parse error caught. Treated identically to 6.1. |
| 6.3 | Graceful degradation is active | `add_timestamps.py` and `assemble_output.py` run | `data_dictionary.md` produced with all metadata but placeholder descriptions. Every field shows `[NEEDS CLARIFICATION]`. |
| 6.4 | Graceful degradation is active | `generate_qa_report.py` runs | `qa_report.md` shows completeness = 0% and `Partial` pipeline status. Clear note that LLM content is missing. |

---

### US-7: Glossary Mapping (P3 — Not Demo Day)

**Pipeline Position:** Pre-LLM — Optional step.

**Priority:** P3 — Not required for demo day. Placeholder for future iteration.

**Script:** `glossary_mapper.py` (not built for demo day)

---

### Edge Cases (F2-Specific)

| # | Edge Case | Expected Script Behavior |
|---|---|---|
| 1 | File is not valid JSON | `validate_input.py` catches the error. Returns error with line/column and hint. Pipeline stops. |
| 2 | Valid JSON but missing `table_name`, `source_file`, or `fields` | `validate_input.py` collects all structural errors and reports together. Pipeline stops. |
| 3 | `fields` array is empty | `validate_input.py` returns error: "Schema contains no extractable fields." |
| 4 | A field is missing `field_name` | `validate_input.py` flags the position. Pipeline stops. |
| 5 | `field_name` is empty, whitespace-only, or exceeds 255 characters | `validate_input.py` rejects with specific error per field position. |
| 6 | A field is missing optional metadata | `extract_fields.py` fills deterministic defaults. Pipeline continues. |
| 7 | Duplicate `field_name` values | `extract_fields.py` includes both, logs to `extraction_warnings.json`, uses position fallback for LLM matching. |
| 8 | `field_name` has leading/trailing whitespace or newlines | `extract_fields.py` strips and logs correction. |
| 9 | `nullable` provided as string `"true"`/`"false"` | `extract_fields.py` coerces to boolean and logs correction. |
| 10 | LLM returns malformed JSON | `attach_citations.py` catches parse error. Treats as LLM offline. Produces partial output. |
| 11 | LLM returns fewer objects than input fields | `attach_citations.py` matches by `field_name`. Unmatched fields get placeholders. |
| 12 | LLM returns wrong `field_name` casing | Case-insensitive matching first, then position fallback. `name_case_mismatch` correction logged. |
| 13 | LLM validation fails — retries | F2 retries up to 3 total attempts before degrading to placeholders. |
| 14 | Template file missing | `assemble_output.py` or `generate_qa_report.py` stops with clear error. Setup error, not graceful degradation. |
| 15 | Template has Git merge conflict markers | Script stops with error before any processing. |
| 16 | Stale intermediate files from previous run | Clear `output/intermediate/` before starting new run. See Known Limitations. |
| 17 | Schema with 500+ fields | Chunking is future scope. For demo day, all fields processed in one pass. Noted as limitation. |

---

## 3. Script Specifications

### 3.1 validate_input.py

| Field | Value |
|---|---|
| **Purpose** | Gate the pipeline. Validate that the user-provided file is well-formed JSON with the required schema structure. |
| **Pipeline Position** | Pre-LLM — Step 1 |
| **FRs Satisfied** | FR-001, FR-005 |
| **Input** | `output/intermediate/user_input.json` |
| **Output** | On success: `output/intermediate/validated_schema.json`. On failure: error report listing all detected issues. |

**Core Logic:**

1. Open with `encoding='utf-8-sig'` to strip Unicode BOM if present.
2. Attempt JSON parse. If fails → print user-friendly error with line/column and hint. Stop.
3. Validate root is a JSON object (not array or primitive).
4. Validate `table_name` exists and is non-empty string.
5. Validate `source_file` exists and is non-empty string.
6. Validate `fields` exists and is non-empty array.
7. Validate every element in `fields` is a JSON object.
8. Validate every `field_name` is non-empty string after `.strip()`.
9. Validate no `field_name` exceeds 255 characters after stripping.
10. Collect ALL errors from steps 3–9 and report together.
11. If no errors → write `validated_schema.json` using `safe_write_json()`.

---

### 3.2 extract_fields.py

| Field | Value |
|---|---|
| **Purpose** | Extract field-level metadata into the format expected by F1. Construct `source_path` for each field. |
| **Pipeline Position** | Pre-LLM — Step 2 |
| **FRs Satisfied** | FR-006 |
| **Input** | `output/intermediate/validated_schema.json` |
| **Output** | `output/intermediate/extracted_fields.json` (always). `output/intermediate/extraction_warnings.json` (conditional — only if duplicates found). |

**Core Logic:**

1. For each field, produce exactly 8 keys: `field_name`, `type`, `nullable`, `constraints`, `enums`, `source_path`, `schema_comments`, `table_name`.
2. Fill deterministic defaults for missing optional fields: `type` → `"UNKNOWN"`, `nullable` → `true`, `constraints` → `[]`, `enums` → `[]`, `schema_comments` → `null`.
3. Strip whitespace and newlines from `field_name`. Log corrections if stripped.
4. Coerce string `nullable` to boolean. Log correction if coerced.
5. Construct `source_path`: `"{source_file} → table: {table_name} → field: {field_name}"`.
6. Detect case-sensitive duplicate `field_name` values. Log to `extraction_warnings.json` if found. Do NOT create this file if no duplicates.
7. Write `extracted_fields.json` — exactly 8 keys per record, always.

---

### 3.3 attach_citations.py

| Field | Value |
|---|---|
| **Purpose** | Validate LLM output, merge with extracted metadata, handle failures gracefully. |
| **Pipeline Position** | Post-LLM — Step 1 |
| **FRs Satisfied** | FR-008 (script side), FR-009 (script side) |
| **Input** | `output/intermediate/extracted_fields.json` + `output/intermediate/llm_output.json` (optional) |
| **Output** | `output/intermediate/merged_fields.json` — 14 keys per field. |

**Core Logic:**

1. Read `extracted_fields.json`.
2. Attempt to read and parse `llm_output.json`. If missing or unparseable → all fields get placeholders. Continue.
3. If LLM output available, validate structure: root must be JSON array; each element must be object with all 5 keys; `confidence` must be `"High"`, `"Medium"`, or `"Low"`; `clarification_flag` must be boolean; `evidence_refs` must be non-empty array.
4. Root-level failure → all placeholders. Per-field failure → only affected field gets placeholders.
5. Match LLM output to input fields by `field_name` (case-insensitive). For duplicates, fall back to position matching.
6. Apply corrections: `clarification_flag_override`, `name_case_mismatch`, `description_over_limit`, `order_mismatch`, `duplicate_name_in_output`, `field_name_whitespace_stripped`, `field_name_newline_stripped`, `nullable_type_coerced`.
7. F2 owns retry logic — max 2 retries (3 total attempts). If all fail → graceful degradation.
8. Write `merged_fields.json` — 14 keys per field.

---

### 3.4 add_timestamps.py

| Field | Value |
|---|---|
| **Purpose** | Add a single `last_verified` timestamp to every field record. |
| **Pipeline Position** | Post-LLM — Step 2 |
| **FRs Satisfied** | FR-010 |
| **Input** | `output/intermediate/merged_fields.json` |
| **Output** | `output/intermediate/timestamped_fields.json` — 15 keys per field. |

**Core Logic:**

1. Generate one timestamp for the run: `datetime.now(timezone.utc).strftime('%Y-%m-%dT%H:%M:%SZ')`.
2. Do NOT use `datetime.now()` without timezone. Do NOT use `datetime.utcnow()` (deprecated in Python 3.12+).
3. Add `last_verified` to every field. Same value for all fields in the run.

---

### 3.5 assemble_output.py

| Field | Value |
|---|---|
| **Purpose** | Populate the data dictionary template to produce `data_dictionary.md`. |
| **Pipeline Position** | Post-LLM — Step 3 |
| **FRs Satisfied** | FR-012 |
| **Input** | `output/intermediate/timestamped_fields.json` + `assets/data_dictionary_template.md` |
| **Output** | `output/data_dictionary.md` |

**Core Logic:**

1. Use `check_template_exists()` to verify template. Stop with error if missing.
2. Scan template for Git merge conflict markers. Stop with error if found.
3. Substitute 3 header placeholders using `.replace()` in fixed order: `{table_name}`, `{source_file}`, `{generation_date}`. Do NOT use `str.format()`.
4. Substitution happens BEFORE field data insertion.
5. Apply `sanitize_for_markdown()` to ALL values in Markdown table cells.
6. Apply display conversions: `[]` → `None`, `null` → `None`, `""` → `None`.
7. Render enums separated by semicolons (not commas).
8. Render `evidence_refs` as numbered list.
9. Prepend `[NEEDS CLARIFICATION]` to description if `clarification_flag` = `true`.
10. Scan assembled output for unreplaced placeholders. Log warning if found. Pipeline continues.

---

### 3.6 generate_qa_report.py

| Field | Value |
|---|---|
| **Purpose** | Produce QA report summarizing coverage, confidence distribution, corrections, and warnings. |
| **Pipeline Position** | Post-LLM — Step 4 (final) |
| **FRs Satisfied** | FR-013 |
| **Input** | `output/intermediate/timestamped_fields.json` + `output/intermediate/extraction_warnings.json` (if exists) + `assets/qa_report_template.md` |
| **Output** | `output/qa_report.md` |

**Core Logic:**

1. Use `check_template_exists()` to verify template. Stop with error if missing.
2. Scan template for Git merge conflict markers. Stop with error if found.
3. Compute Coverage Statistics: coverage = (fields with `merge_status` = `"matched"` / total) × 100.
4. Compute Confidence Distribution: count `"High"`, `"Medium"`, `"Low"`, `"N/A"`. Verify sum = total fields. Log integrity warning if not.
5. List Fields Requiring Clarification: all where `clarification_flag` = `true`.
6. List Merge Corrections (only if corrections exist).
7. List Warnings (only if warnings exist — extraction warnings, LLM validation issues, integrity warnings).
8. Compute Processing Notes: status is `Complete`, `Complete with warnings`, or `Partial`.
9. Use exact section names from F3's template (authoritative per FR-F3-005).

---

### 3.7 utils.py (Shared Utility Module)

Not a pipeline script. Imported by all 6 scripts. Contains:

- Placeholder constants (`PLACEHOLDER_DESCRIPTION`, `PLACEHOLDER_CONFIDENCE`, etc.)
- Encoding constants (`INPUT_ENCODING = "utf-8-sig"`, `OUTPUT_ENCODING = "utf-8"`, `TEMPLATE_ENCODING = "utf-8"`)
- `sanitize_for_markdown(value)` — escapes `|`, `*`, `` ` ``, `\`, `[`, `]`
- `display_value_for_markdown(value)` — converts empty/null values for table cells
- `safe_write_json(data, filepath)` — wraps `json.dump` in try/except
- `safe_write_text(content, filepath)` — wraps file write in try/except
- `ensure_output_dirs()` — creates `output/` and `output/intermediate/` if needed
- `check_template_exists(filepath)` — stops with error if template missing
- `check_git_conflict_markers(content, filepath)` — stops with error if markers found

---

### 3.8 glossary_mapper.py (P3 — Not Demo Day)

Not built for demo day. See US-7.

---

## 4. Requirements

### 4.1 Functional Requirements

#### FR-001: Schema Input Acceptance
The system MUST accept JSON schema files as input. JSON only for demo day.

#### FR-005: Input Validation
ALL input validation MUST be handled by `validate_input.py` before any LLM processing. Two-tier approach: Tier 1 (blocking) for structural errors, Tier 2 (non-blocking) for missing optional fields.

#### FR-006: Field Extraction
The system MUST extract all fields and construct `source_path` for each in format: `"{source_file} → table: {table_name} → field: {field_name}"`. Missing optional metadata filled with deterministic defaults.

#### FR-008: Citation Attachment (Script Side)
The script MUST attach `source_path` to every field. `evidence_refs` content comes from the LLM — the script passes it through.

#### FR-009: Confidence Score Assignment (Script Side)
The script MUST include the LLM's confidence score in the final output. If LLM did not return a score, set `confidence` to `"N/A"`.

#### FR-010: Timestamps
The system MUST include a `last_verified` ISO 8601 UTC timestamp for each field. One timestamp per run — all fields share the same value.

#### FR-011: Glossary Mapping — Script Side (P3 — Deferred)
Not built for demo day.

#### FR-012: Data Dictionary Output
The system MUST produce `data_dictionary.md` by populating F3's template. Markdown only for demo day.

#### FR-013: QA Report Output
The system MUST produce `qa_report.md` with the 6 sections defined by F3's template (authoritative per FR-F3-005): Coverage Statistics, Confidence Distribution, Fields Requiring Clarification, Merge Corrections, Warnings, Processing Notes.

### 4.2 F2-Specific Requirements

#### FR-F2-001: LLM Output Validation
`attach_citations.py` MUST validate count, field_name matching, confidence enum, flag type, and evidence_refs type. F2 owns retry logic — max 2 retries (3 total attempts).

#### FR-F2-002: Graceful Degradation
If the LLM is offline or returns invalid output after all retries, all scripts MUST still run and produce both deliverables with placeholder values.

#### FR-F2-003: Error Collection
`validate_input.py` MUST collect and report all structural errors together.

#### FR-F2-004: Intermediate File Convention
Scripts MUST communicate via JSON files in `output/intermediate/`. File names: `validated_schema.json`, `extracted_fields.json`, `extraction_warnings.json` (conditional), `llm_output.json` (from F1), `merged_fields.json`, `timestamped_fields.json`. Final outputs: `output/data_dictionary.md`, `output/qa_report.md`.

#### FR-F2-005: Standard Library Only
No third-party packages. Python 3.11+ standard library only.

#### FR-F2-006: File Encoding
Input files: `utf-8-sig`. Output files: `utf-8`. Template files: `utf-8`. Never rely on Python's default encoding.

### 4.3 Input Contract (User → F2)

**Required top-level keys:**

| Key | Type | Description |
|---|---|---|
| `table_name` | string | Name of the database table |
| `source_file` | string | Name of the schema file |
| `fields` | array (non-empty) | List of field objects |

**Required per field:**

| Key | Type | Description |
|---|---|---|
| `field_name` | string | Non-empty, ≤255 characters after stripping |

**Optional per field (deterministic defaults if missing):**

| Key | Type | Default | Description |
|---|---|---|---|
| `type` | string | `"UNKNOWN"` | Data type |
| `nullable` | boolean | `true` | Whether field can be empty |
| `constraints` | array of strings | `[]` | Database rules |
| `enums` | array | `[]` | Fixed allowed values |
| `schema_comments` | string or null | `null` | Human-written notes |

### 4.4 Output Contract (F2 → F1)

F2 sends a JSON array of 8-key field objects to the LLM. See data-model.md Section 2 for full schema.

### 4.5 Input Contract (F1 → F2)

F2 receives a JSON array from the LLM — one object per input field with exactly 5 keys: `field_name`, `description`, `confidence`, `evidence_refs`, `clarification_flag`. See data-model.md Section 4 for full schema and validation rules.

---

## 5. Pipeline Flow

```
user_input.json
      │
      ▼
validate_input.py ──── [FAIL] ──→ Error report (all issues listed)
      │ [PASS]
      ▼
extract_fields.py ──→ extracted_fields.json
      │               (+ extraction_warnings.json if duplicates)
      ▼
┌─────────────────────────────────────────┐
│  F1 — SKILL.md (LLM)                   │
│  Returns llm_output.json               │
│  (may fail → US-6 graceful degradation)│
└─────────────────────────────────────────┘
      │
      ▼
attach_citations.py ──→ merged_fields.json
      │
      ▼
add_timestamps.py ──→ timestamped_fields.json
      │
      ▼
assemble_output.py ──→ data_dictionary.md  ← uses assets/data_dictionary_template.md
      │
      ▼
generate_qa_report.py ──→ qa_report.md  ← uses assets/qa_report_template.md
```

---

## 6. Boundaries — What F2 Does NOT Do

| Responsibility | Owned By |
|---|---|
| Generating field descriptions | F1 (SKILL.md) |
| Assigning confidence scores | F1 (SKILL.md) |
| Designing output templates | F3 (Assets) |
| Creating sample_schema.json | F3 (Assets) |
| Creating gold standard examples | F3 (Assets) |

---

## 7. Known Limitations

| Limitation | Mitigation |
|---|---|
| Stale intermediate files between runs | Clear `output/intermediate/` before each new run |
| Case-variant duplicate field names not flagged at extraction | Post-handoff recommendation: add case-insensitive duplicate detection |
| Large enum arrays rendered in full | Post-handoff: add optional truncation |
| 500+ field schemas not chunked | Future scope — demo day is 25 fields |
| Percentage rounding may not sum to 100% | Cosmetic only |

---

## 8. Success Criteria

| SC ID | What We Measure | Threshold | How We Measure |
|---|---|---|---|
| SC-F2-001 | Extraction Completeness — all fields present in extracted metadata | 100% | Automated: compare input count to `extracted_fields.json` count |
| SC-F2-002 | Validation Accuracy — invalid inputs caught and reported | 100% of Tier 1 errors caught | Manual: test with known-bad inputs |
| SC-F2-003 | Citation Coverage — every field has `source_path` | 100% | Automated: check output |
| SC-F2-004 | Timestamp Coverage — every field has `last_verified` | 100% | Automated: check output |
| SC-F2-005 | QA Report Accuracy — stats match actual counts | Exact match | Manual: cross-check |
| SC-F2-006 | Graceful Degradation — partial output when LLM offline | Both deliverables produced | Manual: simulate LLM failure |
| SC-F2-007 | Determinism — same input → identical output | 100% match (excluding timestamps) | Automated: run 3x, diff outputs |
| SC-F2-008 | LLM Output Validation — mismatches detected | All mismatches caught | Manual: test with mismatched output |

---

## 9. Test Data

- **Dataset:** UCI Default of Credit Card Clients (Kaggle) — 25 fields
- **Test Input:** `assets/sample_schema.json` — created by F3
- **Why this dataset:** Mix of clear fields (`AGE`, `LIMIT_BAL`), ambiguous fields (`PAY_0`), cryptic enums (`SEX`), numbered series (`BILL_AMT1`–`BILL_AMT6`), and oddly formatted names (`default.payment.next.month`)

---

## 10. Open Items

| # | Item | Status |
|---|---|---|
| 1 | F3 template `data_dictionary_template.md` | ⏳ PENDING — F3 deliverable |
| 2 | F3 template `qa_report_template.md` | ⏳ PENDING — F3 deliverable |
| 3 | F3 test data `sample_schema.json` | ⏳ PENDING — F3 deliverable |
| 4 | SC-F2-005 and SC-F2-008 thresholds | ⏳ TBD after baseline |
| 5 | Chunking for 500+ field schemas | 📋 FUTURE SCOPE |
| 6 | YAML and DDL input support | 📋 FUTURE SCOPE |
| 7 | CSV output format | 📋 FUTURE SCOPE |
| 8 | Glossary mapper detailed design | 📋 FUTURE SCOPE — P3 |
