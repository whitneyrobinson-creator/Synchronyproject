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

> **Pipeline Flow Reference:** Each user story notes its position in the pipeline so the reader understands when it executes relative to the LLM (F1).
>
> ```
> [Pre-LLM]  US-1 → US-2 → [F1/LLM generates descriptions] → [Post-LLM] US-3 → US-4 → US-5
>                                                                          ↕
>                                                               US-6 (fallback if LLM fails)
> US-7 (P3, future scope — runs pre-LLM if glossary provided)
> ```

---

### US-1: Parse and Validate a Schema File (P1)

**Pipeline Position:** Pre-LLM — Step 1. This is the first thing that runs. Nothing else executes until validation passes.

**As a** documentation team member,
**I want** the system to accept my JSON schema file, validate that it's well-formed and structurally correct, and tell me about all problems at once if something is wrong,
**So that** I can fix any issues in one pass before the pipeline runs.

**Priority:** P1 — Required for demo day.

**Script:** `validate_input.py`

**What this tests:** The scripts' ability to accept JSON input, enforce structural requirements, and provide clear, actionable error messages.

**Acceptance Scenarios:**

| # | Given | When | Then |
|---|---|---|---|
| 1.1 | A valid JSON schema file with `table_name`, `source_file`, and a non-empty `fields` array where every field has a `field_name` | The user provides the file to the skill | `validate_input.py` passes with no errors. Writes `validated_schema.json`. Pipeline proceeds to US-2 (extraction). |
| 1.2 | A JSON file missing `table_name` and containing two fields without `field_name` | The user provides the file to the skill | `validate_input.py` collects all three errors and returns them in a single error report. Pipeline does not proceed. |
| 1.3 | A file that is not valid JSON (e.g., trailing commas, missing brackets) | The user provides the file to the skill | `validate_input.py` catches the parse error immediately and returns a specific error message. Pipeline does not proceed. |
| 1.4 | A YAML or DDL file instead of JSON | The user provides the file to the skill | `validate_input.py` returns a clear error: "Only JSON schema files are supported in this version. Please convert to JSON." |
| 1.5 | A valid JSON file where `fields` is an empty array | The user provides the file to the skill | `validate_input.py` returns error: "Schema contains no extractable fields." Pipeline does not proceed. |

---

### US-2: Extract Field Metadata from a Valid Schema (P1)

**Pipeline Position:** Pre-LLM — Step 2. Runs after validation passes (US-1). Produces the metadata that gets sent to the LLM (F1) for description generation.

**As a** documentation team member,
**I want** the system to extract all field-level metadata from my schema file — names, types, constraints, enums, comments — and construct a traceable source path for each field,
**So that** the LLM has everything it needs to write accurate descriptions, and every field is traceable back to its origin.

**Priority:** P1 — Required for demo day.

**Script:** `extract_fields.py`

**What this tests:** The scripts' ability to parse a JSON schema, extract all available metadata per field, construct source paths, and handle fields with missing optional data gracefully.

**Acceptance Scenarios:**

| # | Given | When | Then |
|---|---|---|---|
| 2.1 | A valid schema with 25 fields, all with complete metadata (type, nullable, constraints, enums, schema_comments) | `extract_fields.py` processes `validated_schema.json` | Output `extracted_fields.json` contains 25 objects, each with all 8 fields from the F2 → F1 contract. `source_path` is constructed as `{source_file} → table: {table_name} → field: {field_name}`. |
| 2.2 | A valid schema where some fields are missing `type`, `enums`, or `schema_comments` | `extract_fields.py` processes `validated_schema.json` | Missing optional fields are set to `null` in the output. All fields are still included. Pipeline continues — the LLM handles ambiguity. |
| 2.3 | A valid schema with duplicate `field_name` values | `extract_fields.py` processes `validated_schema.json` | Both entries are included in the output. Duplicates are logged to `extraction_warnings.json` for the QA report. |

**What happens next:** `extracted_fields.json` is sent to F1 (SKILL.md). The LLM reads the metadata, generates descriptions, assigns confidence scores, and returns `llm_output.json`. US-3 picks up after the LLM responds.

---

### US-3: Validate and Merge LLM Output with Citations (P1)

**Pipeline Position:** Post-LLM — Step 1. This runs **after the LLM (F1) has generated descriptions, confidence scores, and evidence references** and returned them as `llm_output.json`. This is where F2 quality-checks the LLM's work and merges it with the extracted metadata.

**As a** documentation team member,
**I want** the system to validate that the LLM returned output for every field I sent it, match each response back to the correct field, merge the LLM's descriptions and scores with my original metadata and source citations, and clearly flag any fields the LLM missed,
**So that** I can trust the data dictionary is complete and every field is traceable.

**Priority:** P1 — Required for demo day.

**Script:** `attach_citations.py`

**What this tests:** The scripts' ability to validate LLM output (count check, field_name matching), merge LLM-generated content with script-generated metadata, attach source citations, and handle mismatches gracefully.

**Acceptance Scenarios:**

| # | Given | When | Then |
|---|---|---|---|
| 3.1 | LLM returns a valid JSON array with exactly 25 objects, one per input field, all `field_name` values matching | `attach_citations.py` processes `extracted_fields.json` + `llm_output.json` | Count check passes. Name match passes. `merged_fields.json` contains 25 objects, each with all metadata fields + `description`, `confidence`, `evidence_refs`, `clarification_flag`, and `source_path`. |
| 3.2 | LLM returns 23 objects instead of 25 (2 fields missing) | `attach_citations.py` processes the files | Count mismatch logged. The 23 matched fields are merged normally. The 2 unmatched fields get: `description` = "[LLM did not return a description]", `confidence` = "N/A", `evidence_refs` = "LLM output unavailable", `clarification_flag` = true. Mismatch logged for QA report. |
| 3.3 | LLM returns a `field_name` that doesn't match any input field | `attach_citations.py` processes the files | Orphaned LLM output is logged and ignored. The unmatched input field gets placeholder values. Mismatch noted for QA report. |
| 3.4 | LLM returns valid output but the schema has duplicate `field_name` values | `attach_citations.py` processes the files | Exact `field_name` match attempted first. For duplicates, falls back to array position matching. Both fields are merged. |
| 3.5 | `source_path` from `extracted_fields.json` is attached to every merged field | `attach_citations.py` completes | Every object in `merged_fields.json` includes `source_path` from the extraction step (F2 owns this, not the LLM). |

---

### US-4: Timestamp and Assemble the Data Dictionary (P1)

**Pipeline Position:** Post-LLM — Step 2. Runs **after LLM output has been validated and merged** (US-3). This step adds timestamps and produces the final deliverable.

**As a** documentation team member,
**I want** the system to add a `last_verified` timestamp to every field and then assemble all the data — metadata, descriptions, citations, scores, flags, and timestamps — into a single, formatted data dictionary document,
**So that** I receive one audit-ready deliverable I can review and share.

**Priority:** P1 — Required for demo day.

**Scripts:** `add_timestamps.py` → `assemble_output.py`

**What this tests:** The scripts' ability to apply consistent timestamps and populate the F3 template into a well-formatted, complete Markdown document.

**Acceptance Scenarios:**

| # | Given | When | Then |
|---|---|---|---|
| 4.1 | `merged_fields.json` contains 25 fully merged field objects | `add_timestamps.py` runs | `timestamped_fields.json` contains all 25 objects, each with a new `last_verified` field in ISO 8601 format. All fields share the same timestamp (one per run). |
| 4.2 | `timestamped_fields.json` is ready and `data_dictionary_template.md` exists in F3 assets | `assemble_output.py` runs | `data_dictionary.md` is produced. Every field includes: field_name, type (or "Not specified"), nullable (or "Not specified"), constraints (or "None"), enums (or "None"), description, source_path, evidence_refs, confidence, clarification_flag (prepends `[NEEDS CLARIFICATION]` if true), and last_verified. |
| 4.3 | A field has `confidence` = "N/A" and `clarification_flag` = true (LLM missed it) | `assemble_output.py` runs | The field appears in the data dictionary with `[NEEDS CLARIFICATION]` prepended to the description placeholder. No data is silently dropped. |
| 4.4 | The skill is run twice on the same input | Both runs complete | Outputs are identical except for the `last_verified` timestamp, which reflects the time of each run. |

---

### US-5: Generate the QA Report (P1)

**Pipeline Position:** Post-LLM — Step 3. This is the **final step** in the pipeline. Runs after the data dictionary has been assembled (US-4). Summarizes the entire run.

**As a** documentation team member,
**I want** the system to produce a QA report alongside the data dictionary showing coverage stats, flagged items, and any warnings from the pipeline run,
**So that** I know exactly what needs my attention and can verify completeness at a glance.

**Priority:** P1 — Required for demo day.

**Script:** `generate_qa_report.py`

**What this tests:** The scripts' ability to count, categorize, and report on the completeness and quality of the generated data dictionary.

**Acceptance Scenarios:**

| # | Given | When | Then |
|---|---|---|---|
| 5.1 | A complete data dictionary with 25 fields, all with descriptions and citations | `generate_qa_report.py` runs | `qa_report.md` shows: total fields = 25, fields with descriptions = 25, fields with citations = 25, fields with confidence scores = 25, fields flagged [NEEDS CLARIFICATION] = count of Low confidence fields, completeness = 100%. |
| 5.2 | A partial data dictionary where 2 fields have placeholder descriptions (LLM missed them) | `generate_qa_report.py` runs | `qa_report.md` shows: fields with descriptions = 23, completeness = 92%. The 2 missing fields are listed by name in a "Fields Missing LLM Output" section. |
| 5.3 | A partial data dictionary produced during graceful degradation (LLM was offline — no descriptions at all) | `generate_qa_report.py` runs | `qa_report.md` shows: fields with descriptions = 0, completeness = 0%, and a note: "LLM output was unavailable. Descriptions, confidence scores, and clarification flags are missing." |
| 5.4 | `extraction_warnings.json` exists with duplicate field name warnings | `generate_qa_report.py` runs | `qa_report.md` includes a "Warnings" section listing the duplicate field names from the extraction step. |
| 5.5 | LLM output had count or name mismatches (logged by attach_citations.py) | `generate_qa_report.py` runs | `qa_report.md` includes a "LLM Output Validation" section noting the specific mismatches detected. |

---

### US-6: Graceful Degradation — LLM Offline or Failed (P1)

**Pipeline Position:** Fallback path. This activates when the LLM (F1) is **offline, unreachable, or returns malformed/unparseable output**. The pre-LLM scripts (US-1, US-2) run normally. The post-LLM scripts (US-3, US-4, US-5) run in degraded mode.

**As a** documentation team member,
**I want** the system to still produce a useful partial data dictionary and QA report even when the LLM is unavailable,
**So that** I have all the raw metadata organized and ready — and I can manually write descriptions or re-run the skill later when the LLM is back online.

**Priority:** P1 — Required for demo day. This is a Constitution principle (Graceful Degradation).

**Scripts:** All scripts still run. `attach_citations.py` detects the failure and sets all LLM fields to placeholders.

**What this tests:** The system's ability to degrade gracefully without crashing, losing data, or producing corrupted output.

**Acceptance Scenarios:**

| # | Given | When | Then |
|---|---|---|---|
| 6.1 | LLM is offline — `llm_output.json` does not exist | `attach_citations.py` runs | `llm_available` set to false. All fields get: `description` = "[LLM did not return a description]", `confidence` = "N/A", `evidence_refs` = "LLM output unavailable", `clarification_flag` = true. Pipeline continues. |
| 6.2 | LLM returns malformed JSON (unparseable) | `attach_citations.py` attempts to parse `llm_output.json` | Parse error caught. Treated identically to LLM offline (scenario 6.1). Pipeline continues. |
| 6.3 | Graceful degradation is active | `add_timestamps.py` and `assemble_output.py` run | `data_dictionary.md` is produced with all metadata (field_name, type, nullable, constraints, enums, source_path, last_verified) but placeholder descriptions and no confidence scores. Every field shows `[NEEDS CLARIFICATION]`. |
| 6.4 | Graceful degradation is active | `generate_qa_report.py` runs | `qa_report.md` shows completeness = 0% and includes a clear note: "LLM output was unavailable. All descriptions, confidence scores, and clarification flags are placeholders. Re-run the skill when the LLM is available, or manually add descriptions." |

---

### US-7: Glossary Mapping (P3)

**Pipeline Position:** Pre-LLM — Optional step. Would run **between US-2 (extraction) and the LLM call** if a glossary file is provided. Not implemented for demo day.

**As a** documentation team member,
**I want** the system to map field names to standardized glossary labels when a glossary file is provided,
**So that** the data dictionary uses consistent organizational terminology.

**Priority:** P3 — Not required for demo day. Placeholder for future iteration.

**Script:** `glossary_mapper.py`

**Minimal spec:** If a glossary JSON file is provided alongside the schema, `glossary_mapper.py` performs exact and fuzzy matching of field names to glossary terms. Matched labels are included in the data dictionary output. Ambiguous matches are passed to the LLM (F1) for interpretation. Detailed design deferred.

---

### Edge Cases (F2-Specific)

| # | Edge Case | Expected Script Behavior |
|---|---|---|
| 1 | File is not valid JSON (parse error) | `validate_input.py` catches the error immediately. Returns a specific error message. Pipeline stops. |
| 2 | Valid JSON but missing `table_name` or `fields` | `validate_input.py` collects all structural errors and reports them together. Pipeline stops. |
| 3 | `fields` array is empty | `validate_input.py` returns error: "Schema contains no extractable fields." Pipeline stops. |
| 4 | A field is missing `field_name` | `validate_input.py` flags this as a structural error. Pipeline stops. |
| 5 | A field is missing optional metadata (type, nullable, etc.) | `extract_fields.py` sets missing values to `null`. Pipeline continues. LLM handles ambiguity. |
| 6 | Duplicate `field_name` values in schema | `extract_fields.py` includes both entries. Flags duplicates for QA report. Uses array position as fallback for LLM output matching. |
| 7 | YAML or DDL file provided | `validate_input.py` returns: "Only JSON schema files are supported in this version." |
| 8 | LLM returns malformed JSON | `attach_citations.py` catches parse error. Treats as LLM offline. Produces partial output. |
| 9 | LLM returns fewer objects than input fields | `attach_citations.py` matches by `field_name`. Unmatched fields get placeholder values. Flagged in QA report. |
| 10 | LLM returns a `field_name` that doesn't match any input | `attach_citations.py` logs the orphaned output. Ignores it. Notes mismatch in QA report. |
| 11 | Schema with 500+ fields | Chunking logic is future scope. For demo day, all fields processed in one pass. Noted as a limitation. |
| 12 | Glossary file provided but no matches found | `glossary_mapper.py` (P3) proceeds without mappings. Notes in output that no glossary matches were found. |

---

## 3. Script Specifications

> **Design Principles:** Each script is a single-purpose Python file. Scripts communicate via JSON files written to disk. Each script can be tested independently with known input/output. Scripts are designed to be straightforward and well-documented — the team uses AI-assisted development.

---

### 3.1 validate_input.py

| Field | Value |
|---|---|
| **Purpose** | Validate that the user-provided file is well-formed JSON with the required schema structure. Gate the pipeline — nothing runs if validation fails. |
| **Pipeline Position** | Pre-LLM — Step 1 |
| **User Story** | US-1 |
| **FRs Satisfied** | FR-001, FR-005 |
| **Input** | Raw file path provided by the user |
| **Output** | On success: `validated_schema.json`. On failure: error report listing all detected issues. |

**Core Logic:**

1. Attempt to parse the file as JSON. If parsing fails → return parse error with details. Stop.
2. Check for required top-level keys: `table_name` (string), `fields` (non-empty array).
3. Check each object in `fields` for required key: `field_name` (string).
4. Collect ALL structural errors found in steps 2–3. Return them together.
5. If no errors → write validated JSON to `validated_schema.json`. Return success.

**Error Handling:** Tier 1 — all errors are blocking. Pipeline does not proceed until the user provides a valid file.

---

### 3.2 extract_fields.py

| Field | Value |
|---|---|
| **Purpose** | Parse the validated JSON schema and extract field-level metadata into the format expected by the LLM (F1). Construct `source_path` for each field. |
| **Pipeline Position** | Pre-LLM — Step 2 |
| **User Story** | US-2 |
| **FRs Satisfied** | FR-006 |
| **Input** | `validated_schema.json` (output of validate_input.py) |
| **Output** | `extracted_fields.json` — a JSON array of field metadata objects conforming to the F2 → F1 input contract. Also outputs `extraction_warnings.json` if any issues are detected (e.g., duplicate field names). |

**Core Logic:**

1. Read `validated_schema.json`.
2. For each field in the `fields` array, construct a metadata object:
   - `field_name`: taken directly from input
   - `table_name`: taken from top-level `table_name`
   - `type`: taken from input, or `null` if missing
   - `nullable`: taken from input, or `null` if missing
   - `constraints`: taken from input, or empty array `[]` if missing
   - `enums`: taken from input, or `null` if missing
   - `source_path`: constructed as `"{source_file} → table: {table_name} → field: {field_name}"`
   - `schema_comments`: taken from input, or `null` if missing
3. Check for duplicate `field_name` values. If found, include both but log to `extraction_warnings.json`.
4. Write the array to `extracted_fields.json`.

**Error Handling:** Tier 2 — missing optional fields are set to `null`. Pipeline continues.

---

### 3.3 attach_citations.py

| Field | Value |
|---|---|
| **Purpose** | Receive the LLM's output, validate it (count and field_name matching), and merge it with the extracted metadata and source citations to produce a unified field record for each field. |
| **Pipeline Position** | Post-LLM — Step 1 (runs after F1/LLM returns `llm_output.json`) |
| **User Story** | US-3, US-6 (graceful degradation) |
| **FRs Satisfied** | FR-008 (script side), FR-009 (script side), FR-F2-001, FR-F2-002 |
| **Input** | `extracted_fields.json` (from extract_fields.py) and `llm_output.json` (from F1/LLM). If LLM is offline or returned malformed output, `llm_output.json` may be absent or unparseable. |
| **Output** | `merged_fields.json` — a JSON array where each field object contains both the extracted metadata AND the LLM-generated fields (description, confidence, evidence_refs, clarification_flag, source_path). |

**Core Logic:**

1. Read `extracted_fields.json`.
2. Attempt to read and parse `llm_output.json`.
   - If file is missing or unparseable → set `llm_available = false`. Log the issue. Proceed to step 5.
3. If LLM output is available, validate:
   - Count check: `len(llm_output) == len(extracted_fields)`. Log mismatches.
   - Name match: for each LLM output object, match `field_name` to an extracted field. Use exact match first; if duplicates exist, fall back to array position.
4. For each extracted field, merge:
   - All metadata from `extracted_fields.json`
   - `description`, `confidence`, `evidence_refs`, `clarification_flag` from matched LLM output
   - `source_path` from extracted metadata (script owns this, not the LLM)
5. For any field where LLM output is missing (LLM offline, skipped field, or mismatch):
   - `description`: "[LLM did not return a description]"
   - `confidence`: "N/A"
   - `evidence_refs`: "LLM output unavailable"
   - `clarification_flag`: true
6. Write to `merged_fields.json`.

**Error Handling:** Graceful degradation. LLM failure never stops the pipeline — it produces partial output.

---

### 3.4 add_timestamps.py

| Field | Value |
|---|---|
| **Purpose** | Add a `last_verified` timestamp to every field record. |
| **Pipeline Position** | Post-LLM — Step 2a |
| **User Story** | US-4 |
| **FRs Satisfied** | FR-010 |
| **Input** | `merged_fields.json` (from attach_citations.py) |
| **Output** | `timestamped_fields.json` — same structure as input, with `last_verified` added to each field. |

**Core Logic:**

1. Read `merged_fields.json`.
2. Generate a single timestamp for the entire run: ISO 8601 format (e.g., `2026-04-03T14:30:00Z`).
3. Add `last_verified: "{timestamp}"` to every field object.
4. Write to `timestamped_fields.json`.

**Design Note:** One timestamp per run (not per field) ensures consistency across the entire data dictionary. If the skill is re-run, all fields get the new timestamp.

---

### 3.5 assemble_output.py

| Field | Value |
|---|---|
| **Purpose** | Populate the data dictionary template (from F3) with the timestamped field data to produce the final `data_dictionary.md`. |
| **Pipeline Position** | Post-LLM — Step 2b |
| **User Story** | US-4 |
| **FRs Satisfied** | FR-012 |
| **Input** | `timestamped_fields.json` (from add_timestamps.py) and `data_dictionary_template.md` (from F3 Assets). |
| **Output** | `data_dictionary.md` — the final, human-readable data dictionary. |

**Core Logic:**

1. Read `timestamped_fields.json`.
2. Read `data_dictionary_template.md` from F3 assets.
3. For each field, populate the template with:
   - `field_name`
   - `type` (or "Not specified" if null)
   - `nullable` (or "Not specified" if null)
   - `constraints` (or "None" if empty)
   - `enums` (or "None" if null)
   - `description` (from LLM, or placeholder if unavailable)
   - `source_path`
   - `evidence_refs`
   - `confidence`
   - `clarification_flag` — if true, prepend `[NEEDS CLARIFICATION]` to the description
   - `last_verified`
4. If a glossary label exists (P3), include it.
5. Write the assembled document to `data_dictionary.md`.

**Dependency on F3:** This script requires the template from F3 (Assets). The template defines the exact Markdown structure — this script only fills in the values. Template design is NOT F2's responsibility.

---

### 3.6 generate_qa_report.py

| Field | Value |
|---|---|
| **Purpose** | Produce a QA report summarizing coverage stats, flagged items, and any warnings from the pipeline run. |
| **Pipeline Position** | Post-LLM — Step 3 (final step) |
| **User Story** | US-5 |
| **FRs Satisfied** | FR-013 |
| **Input** | `timestamped_fields.json` (from add_timestamps.py) and `extraction_warnings.json` (from extract_fields.py, if it exists). Also reads `qa_report_template.md` from F3 Assets. |
| **Output** | `qa_report.md` — the QA report. |

**Core Logic:**

1. Read `timestamped_fields.json`.
2. Calculate coverage stats:
   - Total fields
   - Fields with descriptions (description ≠ placeholder)
   - Fields with citations (source_path is present)
   - Fields with confidence scores (confidence ≠ "N/A")
   - Fields flagged [NEEDS CLARIFICATION] (clarification_flag = true)
   - Completeness percentage: (fields with descriptions / total fields) × 100
3. If `extraction_warnings.json` exists, include warnings (e.g., duplicate field names).
4. If LLM output was unavailable, include a note explaining which fields are missing LLM-generated content.
5. Populate `qa_report_template.md` from F3 assets.
6. Write to `qa_report.md`.

**Dependency on F3:** This script requires the QA report template from F3 (Assets).

---

### 3.7 glossary_mapper.py (P3 — Not Demo Day)

| Field | Value |
|---|---|
| **Purpose** | Map field names to standardized glossary labels using a provided glossary file. |
| **Pipeline Position** | Pre-LLM — Optional step (between US-2 and LLM call) |
| **User Story** | US-7 |
| **FRs Satisfied** | FR-011 (script side) |
| **Input** | `extracted_fields.json` and a glossary JSON file. |
| **Output** | Updated `extracted_fields.json` with `glossary_label` added to matched fields. Ambiguous matches flagged for LLM interpretation. |
| **Status** | P3 — Not required for demo day. Detailed design deferred. |

**Minimal Design:**

- Glossary file format: `{"glossary": [{"term": "LIMIT_BAL", "label": "Credit Limit Balance"}, ...]}`
- Exact match on `field_name` → `term`. Assign `glossary_label`.
- No exact match → flag for LLM interpretation (F1).
- Runs between `extract_fields.py` and the LLM step if a glossary is provided.

---

## 4. Requirements

### 4.1 Functional Requirements

#### FR-001: Schema Input Acceptance

- The system MUST accept JSON schema files as input for data dictionary generation.
- **Demo day scope:** JSON only. YAML and DDL support are planned for future iteration.
- The JSON schema must follow the approved flat structure (see Section 4.3).

#### FR-005: Input Validation

- ALL input validation MUST be handled by deterministic script logic (`validate_input.py`) before any LLM processing begins.
- Validation uses a two-tier approach:
  - **Tier 1 (blocking):** Unparseable JSON, missing required structure (`table_name`, non-empty `fields` array, `field_name` per field) → pipeline stops, all errors reported together.
  - **Tier 2 (non-blocking):** Individual fields with missing optional metadata (type, nullable, constraints, enums, schema_comments) → pipeline continues, missing values set to `null`.

#### FR-006: Field Extraction

- The system MUST extract all fields from a schema including: `field_name`, `type`, `nullable`, `constraints`, `enums`, and `schema_comments`.
- The system MUST construct a `source_path` for each field in the format: `"{source_file} → table: {table_name} → field: {field_name}"`.
- Missing optional metadata MUST be set to `null` (not omitted).

#### FR-008: Citation Attachment (Script Side)

- The script MUST attach `source_path` to every field in the final output.
- The script MUST include `evidence_refs` from the LLM output in the final data dictionary.
- **Boundary:** The script determines WHERE the citation points (source_path). The LLM determines WHAT the evidence references say (evidence_refs content).

#### FR-009: Confidence Score Assignment (Script Side)

- The script MUST include the confidence score from the LLM output in the final data dictionary.
- If the LLM did not return a confidence score for a field, the script MUST set confidence to "N/A".
- **Boundary:** The LLM assigns the score (High/Medium/Low). The script passes it through and handles the "N/A" fallback.

#### FR-010: Timestamps

- The system MUST include a `last_verified` timestamp for each field in ISO 8601 format.
- One timestamp per pipeline run — all fields in a single run share the same timestamp.

#### FR-011: Glossary Mapping — Script Side (P3 — Deferred)

- If a glossary JSON file is provided, the script MUST perform exact matching of field names to glossary terms.
- Ambiguous or unmatched fields MUST be flagged for LLM interpretation.
- **Priority:** P3 — Not required for demo day. Detailed design deferred.

#### FR-012: Data Dictionary Output

- The system MUST produce `data_dictionary.md` as the primary output.
- The output MUST be assembled by populating the template from F3 (Assets) with field data.
- **Demo day scope:** Markdown only. CSV output is planned for future iteration.

#### FR-013: QA Report Output

- The system MUST produce `qa_report.md` showing:
  - Total fields
  - Fields with descriptions
  - Fields with citations
  - Fields with confidence scores
  - Fields flagged [NEEDS CLARIFICATION]
  - Completeness percentage
  - Warnings (duplicate field names, LLM issues, validation mismatches)

---

### 4.2 F2-Specific Requirements

#### FR-F2-001: LLM Output Validation

- `attach_citations.py` MUST validate the LLM's output before merging:
  - Count check: number of LLM output objects must equal number of input fields.
  - Name match: each LLM output `field_name` must match an input `field_name`.
  - Mismatches MUST be logged and reported in the QA report.
  - Validation failures MUST NOT stop the pipeline — unmatched fields receive placeholder values.

#### FR-F2-002: Graceful Degradation

- If the LLM is offline or returns malformed output, all scripts MUST still run.
- The system MUST produce a partial data dictionary containing all metadata but no descriptions, confidence scores, or clarification flags.
- The QA report MUST note that LLM-generated content is missing.

#### FR-F2-003: Error Collection

- `validate_input.py` MUST collect all structural errors and report them together (not fail on the first error).
- Exception: if the file is not parseable as JSON at all, the script fails immediately (there is nothing else to validate).

#### FR-F2-004: Intermediate File Convention

- Scripts MUST communicate via JSON files written to disk.
- File naming convention:
  - `validated_schema.json`
  - `extracted_fields.json`
  - `extraction_warnings.json` (if warnings exist)
  - `llm_output.json` (written by F1/SKILL.md)
  - `merged_fields.json`
  - `timestamped_fields.json`
  - `data_dictionary.md` (final output)
  - `qa_report.md` (final output)

---

### 4.3 Input Contract (User → F2)

The user provides a JSON schema file with this structure:

**Required keys** (Tier 1 — must exist or pipeline stops):

| Key | Type | Description |
|---|---|---|
| `table_name` | string | Name of the database table |
| `source_file` | string | Name of the schema file |
| `fields` | array (non-empty) | List of field objects |

**Required per field** (Tier 1 — must exist):

| Key | Type | Description |
|---|---|---|
| `field_name` | string | Column name in the database |

**Optional per field** (Tier 2 — missing is fine, set to null):

| Key | Type | Description |
|---|---|---|
| `type` | string | Data type (INTEGER, VARCHAR, DATE, etc.) |
| `nullable` | boolean | Whether the field can be empty |
| `constraints` | array of strings | Database rules (PRIMARY KEY, FOREIGN KEY, etc.) |
| `enums` | array | Fixed list of allowed values, if any |
| `schema_comments` | string | Human-written notes from the schema |

**Example:**

```json
{
  "table_name": "credit_card_clients",
  "source_file": "sample_schema.json",
  "fields": [
    {
      "field_name": "ID",
      "type": "INTEGER",
      "nullable": false,
      "constraints": ["PRIMARY KEY"],
      "enums": null,
      "schema_comments": "Unique client identifier"
    },
    {
      "field_name": "PAY_0",
      "type": "INTEGER",
      "nullable": true,
      "constraints": [],
      "enums": [-2, -1, 0, 1, 2, 3, 4, 5, 6, 7, 8, 9],
      "schema_comments": null
    }
  ]
}
```
### 4.4 Output Contract (F2 → F1)

F2 sends the LLM a JSON array of field metadata objects:

| Field | Type | Description | Source |
|---|---|---|---|
| `field_name` | string | Column name | Extracted from schema |
| `table_name` | string | Table name | Extracted from schema |
| `type` | string or null | Data type | Extracted from schema |
| `nullable` | boolean or null | Nullable flag | Extracted from schema |
| `constraints` | array | Constraint list | Extracted from schema |
| `enums` | array or null | Enum values | Extracted from schema |
| `source_path` | string | File → table → field path | Constructed by F2 |
| `schema_comments` | string or null | Schema comments | Extracted from schema |

---

### 4.5 Input Contract (F1 → F2)

F2 receives a JSON array from the LLM, one object per input field:

| Field | Type | Description |
|---|---|---|
| `field_name` | string | Echo back for matching |
| `description` | string | Plain-language explanation (≤25 words) |
| `confidence` | string | "High", "Medium", or "Low" |
| `evidence_refs` | string | What evidence the LLM used, with reasoning |
| `clarification_flag` | boolean | Whether field needs human review |

**Validation rules (F2 enforces):**

- Count must match: `len(llm_output) == len(extracted_fields)`
- Every `field_name` in LLM output must match an input `field_name`
- For duplicate field names: fall back to array position matching
- Mismatches are logged and reported in QA report — they do not stop the pipeline

**This resolves F1 Open Items #1 and #2:**

- **F1 Open Item #1:** LLM output format → **Confirmed: JSON array**
- **F1 Open Item #2:** Count validation → **Confirmed: One output per input field, same order. F2 validates and handles mismatches gracefully.**

---

### 4.6 Key Entities

- **Schema File** — The user-provided JSON file describing a database table. One table per file. Follows the structure defined in Section 4.3.
- **Field Metadata Object** — The extracted representation of a single field, containing all metadata from the schema plus the constructed `source_path`. This is what F2 sends to F1.
- **LLM Output Object** — What the LLM returns for a single field: description, confidence, evidence_refs, clarification_flag. This is what F2 receives from F1.
- **Merged Field Record** — The combined object after F2 merges extracted metadata with LLM output, citations, and timestamps. This is what populates the final data dictionary.
- **Intermediate JSON Files** — The files scripts use to pass data between steps (see FR-F2-004). These also serve as debugging artifacts the team can inspect.

---

## 5. Pipeline Flow
User provides JSON schema file │ ▼ validate_input.py ──── [FAIL] ──→ Error report (all issues listed) │ [PASS] │ ▼ extract_fields.py ──→ extracted_fields.json │ (+ extraction_warnings.json if duplicates) │ ▼ ┌─────────────────────────────────────────────────────────────┐ │ F1 — SKILL.md (LLM) │ │ Receives extracted_fields.json │ │ Generates descriptions, confidence scores, evidence_refs │ │ Returns llm_output.json │ │ (may fail or be offline → US-6 graceful degradation) │ └─────────────────────────────────────────────────────────────┘ │ ▼ attach_citations.py ──→ merged_fields.json │ (validates LLM output, merges, handles failures) ▼ add_timestamps.py ──→ timestamped_fields.json │ ▼ assemble_output.py ──→ data_dictionary.md ← uses F3 template │ ▼ generate_qa_report.py ──→ qa_report.md ← uses F3 template


**F2's scope:** Everything in this diagram EXCEPT the F1 box. F2 owns all scripts, all intermediate files, and both final outputs. F1 owns the LLM reasoning. F3 owns the templates.

---

## 6. Boundaries — What F2 Does NOT Do

| Responsibility | Owned By | Why Not F2 |
|---|---|---|
| Generating field descriptions | F1 (SKILL.md) | Requires LLM reasoning |
| Assigning confidence scores (High/Medium/Low) | F1 (SKILL.md) | Requires LLM signal evaluation |
| Interpreting ambiguous fields | F1 (SKILL.md) | Requires LLM judgment |
| Interpreting glossary mappings for ambiguous matches | F1 (SKILL.md) | Requires LLM reasoning |
| Deciding what [NEEDS CLARIFICATION] | F1 (SKILL.md) | Requires LLM assessment |
| Designing the data_dictionary.md template | F3 (Assets) | Template design is a separate concern |
| Designing the qa_report.md template | F3 (Assets) | Template design is a separate concern |
| Creating sample_schema.json for testing | F3 (Assets) | Test data is an asset |

**Key boundary rule:** If the logic can be done without an LLM (parsing, counting, validating, formatting, merging), it belongs in F2. If it requires reasoning, interpretation, or generation, it belongs in F1.

---

## 7. Constitution Constraints Active on F2

| Constitution Principle | F2 Compliance | Notes |
|---|---|---|
| **Accuracy Over Speed** | ✅ Compliant | Scripts validate inputs, validate LLM output, and report coverage stats. No shortcuts that sacrifice correctness. |
| **Graceful Degradation** | ✅ Compliant | If LLM is offline, scripts still parse, extract, validate, timestamp, and produce partial output. QA report notes what's missing. US-6 tests this explicitly. |
| **Simplicity First** | ✅ Compliant | Six small single-purpose Python scripts. No orchestrator, no frameworks, no databases. File-based I/O. SKILL.md is the orchestrator. |
| **Audit-Ready by Default** | ✅ Compliant | Every field includes source_path, evidence_refs, confidence, and last_verified. QA report provides coverage stats. Intermediate JSON files are inspectable debugging artifacts. |
| **Never send raw PII to LLM** | ✅ Compliant | F2 sends only metadata (field names, types, constraints). No actual data values are ever extracted or sent. |
| **Never send API keys or credentials** | ✅ Compliant | No credentials in scripts or intermediate files. |
| **Never send full source code or raw datasets** | ✅ Compliant | Only structured field metadata is sent to the LLM. |
| **Never commit secrets to the repo** | ✅ Compliant | Scripts contain no secrets. |
| **Never opt into LLM provider training** | ✅ Compliant | No opt-in. Scripts do not interact with LLM providers directly. |

**No violations detected.** F2's design is fully aligned with the constitution.

---

## 8. Processing Approach

### Error Handling: Two-Tier System

| Tier | When | Behavior | Example |
|---|---|---|---|
| **Tier 1 (Blocking)** | Input file is structurally invalid | Collect all errors, report together, stop pipeline | Missing `table_name`, empty `fields` array, unparseable JSON |
| **Tier 2 (Non-blocking)** | Individual fields have missing/incomplete data | Set missing values to `null`, continue pipeline | Field missing `type`, `enums`, or `schema_comments` |

**Exception within Tier 1:** If the file is not parseable as JSON at all, fail immediately — there are no further structural checks to perform.

### File I/O Between Scripts

- All scripts communicate via JSON files written to disk.
- Intermediate files serve dual purpose: data handoff AND debugging artifacts.
- File naming follows the convention in FR-F2-004.
- The SKILL.md (F1) tells Claude when to call each script and passes the file paths.

### Graceful Degradation (Constitution Principle 2)

If the LLM goes offline or returns malformed output:

| What Still Works | What's Missing |
|---|---|
| Schema parsing and validation | Field descriptions |
| Field metadata extraction | Confidence scores |
| Source path construction | Evidence references (LLM reasoning) |
| Timestamps | Clarification flags |
| Partial data dictionary assembly | — |
| QA report (noting gaps) | — |

The team can then manually write descriptions using the extracted metadata as input. This is the Constitution's "lose the narrative but keep the data" principle.

### Chunking (Future Scope)

- For demo day (25 fields), all fields are processed in a single LLM pass.
- For schemas with 500+ fields, F2 would need to batch the `extracted_fields.json` into chunks before sending to the LLM. This is noted as future scope and is NOT implemented for demo day.
- The SKILL.md instructions do not change regardless of batch size — chunking is purely F2's responsibility.

---

## 9. Success Criteria

| SC ID | What We Measure | Threshold | How We Measure |
|---|---|---|---|
| SC-F2-001 | **Extraction Completeness** — All fields in the source schema are present in the extracted metadata | 100% of fields extracted | Automated: compare input field count to `extracted_fields.json` count |
| SC-F2-002 | **Validation Accuracy** — Invalid inputs are caught and reported correctly | 100% of Tier 1 errors caught | Manual: test with known-bad inputs |
| SC-F2-003 | **Citation Coverage** — Every field in the final data dictionary includes a `source_path` | 100% of fields have source_path | Automated: check `data_dictionary.md` |
| SC-F2-004 | **Timestamp Coverage** — Every field includes a `last_verified` timestamp | 100% of fields have timestamp | Automated: check `data_dictionary.md` |
| SC-F2-005 | **QA Report Accuracy** — Coverage stats in `qa_report.md` match actual counts in the data dictionary | Stats match exactly | Manual: cross-check report against data dictionary |
| SC-F2-006 | **Graceful Degradation** — When LLM is offline, scripts still produce partial output with QA report | Partial output generated | Manual: simulate LLM failure, verify output |
| SC-F2-007 | **Determinism** — Same input produces identical output across multiple runs | 100% match (excluding timestamps) | Automated: run 3x, diff outputs |
| SC-F2-008 | **LLM Output Validation** — Mismatches between LLM output and input are detected and reported | All mismatches caught | Manual: test with intentionally mismatched LLM output |

---

## 10. Test Data

- **Dataset:** UCI Default of Credit Card Clients (Kaggle)
- **Source File:** `UCI_Credit_Card.csv`
- **Fields:** 25 columns
- **Test Input:** `sample_schema.json` — a JSON schema file created by the team from the CSV, following the structure defined in Section 4.3. This file is an F3 (Assets) deliverable.
- **Why this dataset:** Mix of clear fields (`AGE`, `LIMIT_BAL`), ambiguous fields (`PAY_0`), cryptic numeric enums (`SEX`: 1/2), numbered series (`BILL_AMT1`–`BILL_AMT6`), and an oddly formatted field name (`default.payment.next.month`).
- **Limitation:** This is a proof-of-concept test case. Production schemas may be more complex with nested structures, foreign keys across tables, and proprietary naming conventions.
- **Recommendation:** Use AI-assisted development (e.g., GPT-4o Mini) to generate `sample_schema.json` from the CSV column headers. Team reviews and corrects the output.

---

## 11. Approved Assumptions

| # | Assumption | Mitigation |
|---|---|---|
| 1 | One table per file is sufficient for demo day | Multi-table support is future scope. Team controls the demo input. |
| 2 | LLM may return invalid JSON | Edge case #8: `attach_citations.py` catches parse errors, treats as LLM offline, produces partial output. |
| 3 | 25 fields fits in one LLM pass | Chunking logic is future scope. 25 fields is well within context limits. |
| 4 | `field_name` is unique enough for matching | Duplicate field names use position-based fallback. Duplicates flagged in QA report. |
| 5 | Team can create `sample_schema.json` accurately | AI-assisted creation recommended. Pipeline handles imperfect input gracefully (Tier 2). |
| 6 | Markdown output is sufficient for auditors | .csv and .docx are future scope. Acceptable for academic demo context. |
| 7 | Intermediate JSON files between scripts work in Claude's sandbox | Low risk for 25 fields (tiny files). Consistent naming convention defined in FR-F2-004. |

---

## 12. Open Items

| # | Item | Status |
|---|---|---|
| 1 | F3 template design (`data_dictionary_template.md`) | ⏳ PENDING — F3 team creates this. `assemble_output.py` depends on it. |
| 2 | F3 template design (`qa_report_template.md`) | ⏳ PENDING — F3 team creates this. `generate_qa_report.py` depends on it. |
| 3 | F3 test data (`sample_schema.json`) | ⏳ PENDING — F3 team creates this from UCI CSV. |
| 4 | Success criteria thresholds for SC-F2-005 and SC-F2-008 | ⏳ TBD — after baseline testing |
| 5 | Chunking logic for 500+ field schemas | 📋 FUTURE SCOPE — not demo day |
| 6 | YAML and DDL input support | 📋 FUTURE SCOPE — not demo day |
| 7 | CSV output format | 📋 FUTURE SCOPE — not demo day |
| 8 | Glossary mapper detailed design | 📋 FUTURE SCOPE — P3, not demo day |

---

## 13. Resolved F1 Open Items

F1's spec (Section 12) left two items pending for F2 to confirm. Both are now resolved:

| F1 Open Item | Decision | Rationale |
|---|---|---|
| **#1: LLM output format** — "JSON array recommended, to be confirmed with F2 team" | **Confirmed: JSON array.** | Consistent with all intermediate file formats in F2. Easy to parse, validate, and merge. |
| **#2: Count validation** — "LLM must return exactly one output object per input field, in the same order received, to be confirmed with F2 team" | **Confirmed: One output per input field, same order. F2 validates and handles mismatches gracefully.** | `attach_citations.py` checks count and field_name matching. Mismatches are logged in QA report but do not stop the pipeline. Position-based fallback for duplicate field names. |
