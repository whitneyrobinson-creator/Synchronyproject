# Tasks: F2 — Scripts: Data Dictionary Generation

**Input**: Design documents from `specs/f2-scripts-data-dictionary/`

**Prerequisites**: plan.md (required), project-spec.md (required for user stories), research.md, data-model.md, contracts.md, quickstart.md

**Tests**: Testing tasks are included and marked as human-executed tasks (not agent tasks). The agent builds the scripts; the human validates them.

**Organization**: All implementation tasks belong to User Story 1 (US1 — Generate a Data Dictionary, P1). Testing tasks are grouped separately in the Polish phase. Scripts are grouped by pipeline position: Pre-LLM scripts first (can be tested without F1 or F3), then Post-LLM scripts (some require F3 templates).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1 for all F2 tasks)
- **[HUMAN]**: Task executed by the developer after the agent finishes — not an agent task
- Include exact file paths in descriptions

## Path Conventions

- **Scripts** live at `skills/data-dictionary/scripts/`
- **Spec documents** live in `specs/f2-scripts-data-dictionary/`
- **Project-level docs** live in `specs/` (constitution.md, project-spec.md, data-dictionary-master.md)
- **Assets** (F3 templates, sample schema) live in `assets/` at repository root
- **Runtime output** lives in `skills/data-dictionary/output/` (not checked in)
- **Intermediate files** live in `skills/data-dictionary/output/intermediate/`

**Agent Workspace**: Before execution, copy all F2 planning documents and project-level documents into `skills/data-dictionary/docs/`. The agent reads from `docs/` and builds scripts into `skills/data-dictionary/scripts/`. The `docs/` folder is a working folder — it is not part of the canonical repository structure and should not be checked in. If the `docs/` folder already exists from F1, add the F2 documents to it.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Create the scripts directory, the shared utility module, and verify the output directories exist so the agent has targets to write to.

- [ ] T001 [US1] Create directory `skills/data-dictionary/scripts/` at repository root per `specs/project-spec.md` Repository Structure. If it already exists from a prior setup, confirm it is empty or contains only F1 artifacts.

- [ ] T002 [US1] Create `skills/data-dictionary/scripts/utils.py` — the shared utility module. This file is NOT a pipeline script. It does not run on its own. It holds constants and helper functions that multiple scripts import. It must contain:
  - **Placeholder constants** (exact values from `specs/f2-scripts-data-dictionary/data-model.md` Section 7): `PLACEHOLDER_DESCRIPTION = "[LLM did not return a description]"`, `PLACEHOLDER_CONFIDENCE = "N/A"`, `PLACEHOLDER_EVIDENCE = ["No evidence available — LLM did not process this field"]`, `PLACEHOLDER_CLARIFICATION_FLAG = True`, `PLACEHOLDER_MERGE_STATUS = "placeholder"`.
  - **Encoding constants**: `INPUT_ENCODING = "utf-8-sig"` (strips BOM), `OUTPUT_ENCODING = "utf-8"` (no BOM in output), `TEMPLATE_ENCODING = "utf-8"` (F3 templates are clean UTF-8).
  - **`sanitize_for_markdown(value)`** function: escapes special characters for Markdown table cells — `|` → `\|`, `*` → `\*`, `` ` `` → `` \` ``, `\` → `\\`, `[` → `\[`, `]` → `\]`. Only applied to Markdown output — intermediate JSON retains raw values. Reference: `specs/f2-scripts-data-dictionary/plan.md` Script #5.
  - **`safe_write_json(data, filepath)`** function: wraps `json.dump` in try/except catching `PermissionError` and `OSError`. On failure, prints: `"Cannot write to {filepath}: {error}. Check file permissions and available disk space."` and stops immediately. Reference: `specs/f2-scripts-data-dictionary/plan.md` Error Handling Standards.
  - **`safe_write_text(content, filepath)`** function: same pattern as `safe_write_json` but for writing text/Markdown files using `OUTPUT_ENCODING`.
  - **`ensure_output_dirs()`** function: calls `os.makedirs()` with `exist_ok=True` for both `skills/data-dictionary/output/` and `skills/data-dictionary/output/intermediate/`. Reference: `specs/f2-scripts-data-dictionary/plan.md` Implementation Notes.
  - **`check_template_exists(filepath)`** function: verifies a template file exists at the given path. If missing, prints: `"Template not found at {filepath}. Pipeline cannot produce formatted output without the template. Ensure all F3 asset files are present before running."` and stops. Reference: `specs/f2-scripts-data-dictionary/contracts.md` Contract 3.
  - **`check_git_conflict_markers(content, filepath)`** function: scans file content for `<<<<<<<`, `=======`, `>>>>>>>`. If found, prints: `"Template file at {filepath} contains unresolved Git merge conflict markers. Resolve the conflicts in the template file before running the pipeline."` and stops. Reference: `specs/f2-scripts-data-dictionary/plan.md` Script #5, #6.
  - **`display_value_for_markdown(value)`** function: converts values for Markdown table cells — `[]` (empty list) → `None`, `null`/`None` → `None`, `""` (empty string) → `None`, `true`/`True` → `true`, `false`/`False` → `false`. Only applied to Markdown output. Reference: `specs/f2-scripts-data-dictionary/plan.md` Script #5.

**Checkpoint**: Scripts directory exists. `utils.py` is in place with all shared constants and helpers. Agent can now build individual scripts that import from it.

---

## Phase 2: Foundational — Pre-LLM Scripts (Blocking Prerequisites)

**Purpose**: Build the two scripts that run BEFORE the LLM step. These are the gatekeeper (`validate_input.py`) and the metadata extractor (`extract_fields.py`). They can be fully built and tested without F1 or F3.

**⚠️ CRITICAL**: These scripts must be complete before Post-LLM scripts can be meaningfully tested, because Post-LLM scripts read the files these scripts produce.

- [ ] T003 [US1] Create `skills/data-dictionary/scripts/validate_input.py` — Pre-LLM Step 1. This script validates the user's JSON schema file. It must:
  - Read from `skills/data-dictionary/output/intermediate/user_input.json` using `INPUT_ENCODING` (from `utils.py`) to automatically strip Unicode BOM if present.
  - Call `ensure_output_dirs()` from `utils.py` before any file operations.
  - Check that the file contains valid JSON. If JSON parse fails, print a user-friendly error including the file path, line and column number, and a hint: `"Common causes: trailing commas, missing quotes, single quotes instead of double quotes, or unescaped special characters."`
  - Validate that the root element is a JSON object (not an array or primitive).
  - Validate that `table_name` exists and is a non-empty string.
  - Validate that `source_file` exists and is a non-empty string.
  - Validate that `fields` exists and is a non-empty array.
  - Validate that every element in the `fields` array is a JSON object.
  - Validate that every `field_name` is a non-empty string after `.strip()` — reject fields where `field_name` is `""`, `"   "`, or `null` with error: `"Field at position X has an empty or missing field_name."`
  - Reject fields where `field_name` exceeds 255 characters after stripping, with error: `"Field at position X has a field_name exceeding 255 characters (actual: {length})."`
  - Collect ALL errors and report them at once — do not stop at the first error.
  - If all checks pass, write `skills/data-dictionary/output/intermediate/validated_schema.json` using `safe_write_json()` from `utils.py`.
  - If any check fails, print all errors and stop. Do not write `validated_schema.json`.
  - Reference: `specs/f2-scripts-data-dictionary/plan.md` Script #1, `specs/f2-scripts-data-dictionary/data-model.md` Section 1, `specs/f2-scripts-data-dictionary/contracts.md` Contract 1.

- [ ] T004 [US1] Create `skills/data-dictionary/scripts/extract_fields.py` — Pre-LLM Step 2. This script reads the validated schema and produces a standardized metadata record for every field. It must:
  - Read from `skills/data-dictionary/output/intermediate/validated_schema.json`.
  - For each field, produce an object with exactly **8 keys**: `field_name`, `type`, `nullable`, `constraints`, `enums`, `source_path`, `schema_comments`, `table_name`.
  - Fill defaults for missing optional fields: `type` → `"UNKNOWN"`, `nullable` → `true`, `constraints` → `[]`, `enums` → `[]`, `schema_comments` → `null`.
  - Build `source_path` for each field: `"{source_file} → {table_name} → {field_name}"`. The `→` separator is display-only — never parsed by downstream scripts.
  - Strip leading/trailing whitespace from `field_name` values using `.strip()`. If whitespace was removed, log a correction with type `field_name_whitespace_stripped`.
  - Strip newline characters (`\n`, `\r`) from `field_name` values. If newlines were removed, log a correction with type `field_name_newline_stripped`.
  - If `nullable` is provided as a string (`"true"` or `"false"`, case-insensitive), coerce to the corresponding boolean and log a correction with type `nullable_type_coerced`. Other string values for `nullable` are treated as `null` (unknown), which then defaults to `true`.
  - All corrections (whitespace stripped, newline stripped, nullable coerced) must be logged using the same correction object format as `attach_citations.py` (T005): each correction is an object with 4 keys — `type`, `original`, `corrected`, `reason` — as defined in `specs/f2-scripts-data-dictionary/data-model.md` Section 5 (Correction Object Schema). Store corrections in a `corrections` array on each field. These corrections carry forward through the pipeline and surface in the QA report's Merge Corrections section.
  - Detect duplicate field names (case-sensitive). If duplicates found, write `skills/data-dictionary/output/intermediate/extraction_warnings.json` with the warning structure from `specs/f2-scripts-data-dictionary/data-model.md` Section 3. If no duplicates, do NOT create this file.
  - Write `skills/data-dictionary/output/intermediate/extracted_fields.json` — a JSON array of objects, one per field. Each object has the 8 metadata keys plus a `corrections` array (empty if no corrections were needed for that field).
  - Use `safe_write_json()` from `utils.py` for all file writes.
  - Reference: `specs/f2-scripts-data-dictionary/plan.md` Script #2, `specs/f2-scripts-data-dictionary/data-model.md` Section 2, `specs/f2-scripts-data-dictionary/contracts.md` Contract 1.

**Checkpoint**: Pre-LLM pipeline works. You can feed a JSON schema through `validate_input.py` → `extract_fields.py` and inspect the intermediate files. No F1 or F3 needed.

---

## Phase 3: User Story 1 — Post-LLM Scripts (Priority: P1) 🎯 MVP

**Goal**: Build the four scripts that run AFTER the LLM step. These merge LLM output with extracted metadata, add timestamps, assemble the final data dictionary, and generate the QA report. Scripts 1–2 in this phase (attach_citations, add_timestamps) can be tested without F3. Scripts 3–4 (assemble_output, generate_qa_report) require F3 templates.

**Independent Test**: Run the full pipeline with a fake `llm_output.json` (from `specs/f2-scripts-data-dictionary/quickstart.md` Step 4 Option B). Verify both final deliverables are produced.

### Implementation for User Story 1

- [ ] T005 [US1] Create `skills/data-dictionary/scripts/attach_citations.py` — Post-LLM Step 1. This script validates the LLM's output and merges it with extracted metadata. It must:
  - Read `skills/data-dictionary/output/intermediate/extracted_fields.json` (always required — stop if missing).
  - Read `skills/data-dictionary/output/intermediate/llm_output.json` (optional — if missing, all fields get placeholder values from `utils.py` and pipeline continues).
  - After JSON parse succeeds on `llm_output.json`, validate the structure: root must be a JSON array; each element must be a JSON object with all 5 required keys (`field_name`, `description`, `confidence`, `evidence_refs`, `clarification_flag`); `confidence` must be a string with value `"High"`, `"Medium"`, or `"Low"`; `clarification_flag` must be a boolean; `evidence_refs` must be a non-empty list.
  - Root-level structural failure (not an array, elements not objects): all fields get placeholders.
  - Per-field structural failure (missing keys, wrong types): only the affected field gets placeholders.
  - Log all structural failures as warnings for the QA report's Warnings section.
  - Match LLM output to extracted fields by field name. Use case-insensitive matching. If duplicate field names exist, fall back to position-based matching (1st response → 1st field, 2nd → 2nd).
  - Apply 5 correction types during merge, logging each in the field's `corrections` array:
    - `clarification_flag_override`: confidence is `"Low"` but flag is `false` → flip to `true`.
    - `name_case_mismatch`: LLM returned wrong casing → match case-insensitively, keep original casing.
    - `description_over_limit`: description over 25 words → keep it (soft limit), log word count.
    - `order_mismatch`: LLM returned fields in different order → match by name, not position.
    - `duplicate_name_in_output`: LLM returned two responses for same field → use first match, ignore second.
  - Corrections from T004 (`extracted_fields.json`) are already present in each field's `corrections` array. T005 appends its own corrections to the same array — it does NOT overwrite or replace T004's corrections.
  - For each field, produce a **14-key** merged object (8 from extraction + `description`, `confidence`, `evidence_refs`, `clarification_flag`, `merge_status`, `corrections`).
  - `merge_status` is `"matched"` if LLM output was found, `"placeholder"` if not.
  - Write `skills/data-dictionary/output/intermediate/merged_fields.json`.
  - Use `safe_write_json()` from `utils.py`.
  - Reference: `specs/f2-scripts-data-dictionary/plan.md` Script #3, `specs/f2-scripts-data-dictionary/data-model.md` Sections 4–5, `specs/f2-scripts-data-dictionary/contracts.md` Contract 2, `specs/f2-scripts-data-dictionary/research.md` (LLM Output Validation, Corrections).

- [ ] T006 [US1] Create `skills/data-dictionary/scripts/add_timestamps.py` — Post-LLM Step 2. This script adds a `last_verified` timestamp to every field. It must:
  - Read `skills/data-dictionary/output/intermediate/merged_fields.json`.
  - Generate a single timestamp for the entire run using `datetime.now(timezone.utc).strftime('%Y-%m-%dT%H:%M:%SZ')`. Do NOT use `datetime.now()` without timezone. Do NOT use `datetime.utcnow()` (deprecated in Python 3.12+).
  - Add `last_verified` to every field object — same timestamp for all fields.
  - Each field now has **15 keys**.
  - Write `skills/data-dictionary/output/intermediate/timestamped_fields.json`.
  - Use `safe_write_json()` from `utils.py`.
  - Reference: `specs/f2-scripts-data-dictionary/plan.md` Script #4, `specs/f2-scripts-data-dictionary/data-model.md` Section 6.

- [ ] T007 [P] [US1] Create `skills/data-dictionary/scripts/assemble_output.py` — Post-LLM Step 3. This script produces the final `data_dictionary.md`. It must:
  - Use `check_template_exists()` from `utils.py` to verify `assets/data_dictionary_template.md` exists. If missing, stop with error.
  - Read the template file using `TEMPLATE_ENCODING` from `utils.py`.
  - Use `check_git_conflict_markers()` from `utils.py` to scan the template. If conflict markers found, stop with error. This check runs before any parsing or substitution.
  - Read `skills/data-dictionary/output/intermediate/timestamped_fields.json`.
  - Substitute 3 header placeholders using `.replace()` in this fixed order: (1) `{table_name}`, (2) `{source_file}`, (3) `{generation_date}`. Do NOT use Python's `str.format()` — it throws `KeyError` on unrecognized curly-brace patterns.
  - Substitution happens BEFORE any field data is inserted.
  - Before writing field data to Markdown tables, apply `sanitize_for_markdown()` from `utils.py` to ALL values rendered in table cells.
  - Apply `display_value_for_markdown()` from `utils.py` for value conversions: `[]` → `None`, `null`/`None` → `None`, `""` → `None`, `true` → `true`, `false` → `false`.
  - Render enum values separated by semicolons (not commas, because enum values may contain commas).
  - Render evidence refs as a numbered list in the Evidence & Citations section — one list item per entry in the `evidence_refs` array.
  - For fields where `clarification_flag` is `true`, prepend `[NEEDS CLARIFICATION]` to the description in the output.
  - After all substitution and insertion, scan for any remaining `{table_name}`, `{source_file}`, or `{generation_date}` placeholders. If found, log a warning: `"Unreplaced placeholder detected in output: {placeholder_name}. Check template placeholder names and substitution logic."` Pipeline continues — this is a warning, not a stop.
  - Write `skills/data-dictionary/output/data_dictionary.md` using `safe_write_text()` from `utils.py`.
  - Reference: `specs/f2-scripts-data-dictionary/plan.md` Script #5, `specs/f2-scripts-data-dictionary/contracts.md` Contract 3, `specs/data-dictionary-master.md` Step 7.

- [ ] T008 [P] [US1] Create `skills/data-dictionary/scripts/generate_qa_report.py` — Post-LLM Step 4 (final). This script produces the QA report. It must:
  - Use `check_template_exists()` from `utils.py` to verify `assets/qa_report_template.md` exists. If missing, stop with error.
  - Read the template file using `TEMPLATE_ENCODING` from `utils.py`.
  - Use `check_git_conflict_markers()` from `utils.py` to scan the template. If conflict markers found, stop with error.
  - Read `skills/data-dictionary/output/intermediate/timestamped_fields.json`.
  - Read `skills/data-dictionary/output/intermediate/extraction_warnings.json` if it exists. If it doesn't exist, proceed without warnings.
  - Compute **Coverage Statistics**: `coverage_percentage = (fields with merge_status "matched" / total fields) × 100`. Placeholder fields do not count toward coverage.
  - Compute **Confidence Distribution**: count of `"High"`, `"Medium"`, `"Low"`, and `"N/A"` fields with percentages. After computing, verify that High + Medium + Low + N/A = total field count. If the sum doesn't match, add a warning: `"Pipeline integrity warning: Confidence distribution sum ({sum}) does not match total field count ({total})."`
  - List **Fields Requiring Clarification**: all fields where `clarification_flag` is `true`.
  - List **Merge Corrections**: all corrections from all fields' `corrections` arrays — this includes corrections from both T004 (extraction phase: whitespace stripped, newline stripped, nullable coerced) and T005 (merge phase: flag override, case mismatch, description over limit, order mismatch, duplicate in output). Only include this section if corrections exist.
  - List **Warnings**: extraction warnings (from `extraction_warnings.json`), LLM output validation issues, and pipeline integrity warnings. Only include this section if warnings exist.
  - Compute **Processing Notes**: LLM availability, fields processed count, batching info (`"All fields processed in a single pass"`), run timestamp, and pipeline status. Status is one of: `Complete` (all matched, no warnings), `Complete with warnings` (all matched, but warnings/corrections exist), `Partial` (some or all fields got placeholders).
  - Use the 6 section names exactly as defined in `specs/f2-scripts-data-dictionary/plan.md` QA Report Structure: Coverage Statistics, Confidence Distribution, Fields Requiring Clarification, Merge Corrections, Warnings, Processing Notes.
  - Write `skills/data-dictionary/output/qa_report.md` using `safe_write_text()` from `utils.py`.
  - Reference: `specs/f2-scripts-data-dictionary/plan.md` Script #6 and QA Report Structure, `specs/f2-scripts-data-dictionary/data-model.md` Section 9, `specs/f2-scripts-data-dictionary/contracts.md` Contract 3.

**Checkpoint**: All 6 pipeline scripts are built. Pre-LLM scripts (T003–T004) can be tested now. Post-LLM scripts T005–T006 can be tested with a fake `llm_output.json`. T007–T008 require F3 templates to produce final output.

---

## Phase 4: Polish & Validation

**Purpose**: Human-executed testing and validation tasks. These are NOT agent tasks — the developer runs these after the agent finishes building the scripts.

**⚠️ Before running any test in this phase:** Clear `skills/data-dictionary/output/intermediate/` to prevent stale data from a previous run from contaminating test results. Run: `rm -f skills/data-dictionary/output/intermediate/*.json`. See `specs/f2-scripts-data-dictionary/plan.md` Known Limitations (Stale intermediate files between runs).

- [ ] T009 [HUMAN] [US1] **Pre-LLM smoke test.** Copy the 3-field test schema from `specs/f2-scripts-data-dictionary/quickstart.md` (Preparing Test Input) into `skills/data-dictionary/output/intermediate/user_input.json`. Run `validate_input.py` then `extract_fields.py`. Verify: `validated_schema.json` is created and matches input. Verify: `extracted_fields.json` contains 3 fields, each with exactly 8 keys plus a `corrections` array. Verify: `source_path` is present and correctly formatted for each field. Reference: `specs/f2-scripts-data-dictionary/quickstart.md` Steps 1–3.

- [ ] T010 [HUMAN] [US1] **Bad input rejection test.** Feed `validate_input.py` each of these broken inputs and verify it rejects each one with a clear error message: (1) empty file, (2) valid JSON but missing `table_name`, (3) valid JSON but missing `source_file`, (4) valid JSON but `fields` is empty, (5) a field missing `field_name`, (6) invalid JSON (syntax error). Verify all errors are reported at once (not one at a time). Reference: `specs/f2-scripts-data-dictionary/quickstart.md` Test 3.

- [ ] T011 [HUMAN] [US1] **Graceful degradation test.** Run Steps 1–3 (validate, extract, attach_citations) WITHOUT creating `llm_output.json`. Verify: `merged_fields.json` is created with all fields having placeholder values. Verify: every field has `merge_status: "placeholder"`, `confidence: "N/A"`, `clarification_flag: true`. Then run `add_timestamps.py`. Verify: `timestamped_fields.json` has `last_verified` on every field. If F3 templates are available, run `assemble_output.py` and `generate_qa_report.py` and verify both deliverables are produced with placeholder content and 0% coverage. Reference: `specs/f2-scripts-data-dictionary/quickstart.md` Test 2, `specs/f2-scripts-data-dictionary/research.md` (Placeholders).

- [ ] T012 [HUMAN] [US1] **Happy path test with fake LLM output.** Create a fake `llm_output.json` using the example from `specs/f2-scripts-data-dictionary/quickstart.md` Step 4 Option B. Run the full pipeline (Steps 1–8). Verify: `merged_fields.json` has all fields with `merge_status: "matched"`. Verify: `timestamped_fields.json` has 15 keys per field. If F3 templates are available, verify: `data_dictionary.md` and `qa_report.md` are produced with 100% coverage and no corrections. Reference: `specs/f2-scripts-data-dictionary/quickstart.md` Test 1.

- [ ] T013 [HUMAN] [US1] **Duplicate field name test.** Create a schema with two fields both named `STATUS`. Run Steps 1–3. Verify: `extraction_warnings.json` exists and lists the duplicate. Verify: both fields are present in `extracted_fields.json`. Reference: `specs/f2-scripts-data-dictionary/quickstart.md` Test 4.

- [ ] T014 [HUMAN] [US1] **Determinism test.** Run the same input through the pipeline 3 times. Diff the outputs. Everything should match except `last_verified` timestamps. Reference: `specs/f2-scripts-data-dictionary/quickstart.md` Test 5.

- [ ] T015 [HUMAN] [US1] **25-field end-to-end test.** Run the full pipeline against `assets/sample_schema.json` (25-field UCI Credit Card dataset, created by F3). Verify: 25 field records at every intermediate stage. Verify: field count in QA report matches 25. This test requires F3's sample schema and templates. If F3 is not yet complete, defer this task. Reference: `specs/f2-scripts-data-dictionary/plan.md` Technical Context (Testing).

- [ ] T016 [HUMAN] [US1] **Constitution compliance review.** Verify all 6 scripts against `specs/constitution.md`: no raw PII processed, no credentials, no full source code sent, metadata only, no third-party packages (`pip install` = plan violation), standard library only, file-based I/O only, no database, no network calls. Verify against `specs/f2-scripts-data-dictionary/plan.md` Constitution Check — all 9 items should pass. Reference: `specs/constitution.md` Section 3 (Never-Ever Rules).

- [ ] T017 [HUMAN] [US1] **Run quickstart.md validation.** Walk through `specs/f2-scripts-data-dictionary/quickstart.md` Steps 0–8 as if you were a new team member. Verify every step works as documented. Flag any discrepancies between the quickstart and the actual script behavior. Reference: `specs/f2-scripts-data-dictionary/quickstart.md`.

- [ ] T018 [HUMAN] [US1] **Git conflict marker detection test.** Add a `<<<<<<<` marker to a copy of `assets/data_dictionary_template.md`. Run `assemble_output.py`. Verify: script stops with error message `"Template file at {path} contains unresolved Git merge conflict markers. Resolve the conflicts in the template file before running the pipeline."` Repeat with a copy of `assets/qa_report_template.md` and `generate_qa_report.py`. Verify same behavior. Remove the modified copies after testing. Reference: `specs/f2-scripts-data-dictionary/plan.md` Script #5, #6.

**Checkpoint**: All scripts pass smoke tests. Graceful degradation works. Bad input is rejected. Determinism confirmed. Git conflict detection verified. Constitution compliant. F2 is complete (pending F3 templates for full end-to-end).

---

## Dependencies & Execution Order

- **Setup (Phase 1)**: No dependencies — can start immediately. If F1 has already been built, the `skills/data-dictionary/` directory already exists.
- **Foundational (Phase 2)**: Depends on Phase 1 (`utils.py` must exist before scripts can import from it). BLOCKS Phase 3 — Post-LLM scripts read files produced by Pre-LLM scripts.
- **User Story 1 (Phase 3)**: Depends on Phase 2 completion. T005 reads `extracted_fields.json` produced by T004. T006 reads `merged_fields.json` produced by T005. T007 and T008 read `timestamped_fields.json` produced by T006.
- **Polish & Validation (Phase 4)**: Depends on Phase 3 completion. Human-executed — not agent tasks.

### Within Phase 1 (Setup)

- T001 must complete before T002 (directory must exist before creating files in it)

### Within Phase 2 (Foundational)

- T003 and T004 are sequential: T004 reads the output of T003
- Both write to different files, but T004 depends on T003's output existing for testing

### Within Phase 3 (User Story 1)

- T005 depends on T004 (reads `extracted_fields.json`)
- T006 depends on T005 (reads `merged_fields.json`)
- T007 depends on T006 (reads `timestamped_fields.json`) AND requires F3 template (`assets/data_dictionary_template.md`)
- T008 depends on T006 (reads `timestamped_fields.json`) AND requires F3 template (`assets/qa_report_template.md`)
- T007 and T008 are marked [P] — they both read `timestamped_fields.json` but write to different output files (`data_dictionary.md` and `qa_report.md`). They can be built in parallel or sequentially.

### Within Phase 4 (Polish & Validation)

- T009 (Pre-LLM smoke test) should run first — validates the foundation
- T010 (bad input test) can run in parallel with T009
- T011 (graceful degradation) and T012 (happy path) depend on T009 passing
- T013 (duplicate test) can run anytime after T009
- T014 (determinism) depends on T012 passing
- T015 (25-field test) depends on F3's `assets/sample_schema.json` and templates existing
- T016, T017, and T018 can run in parallel with other Phase 4 tasks

### Cross-Feature Dependencies

- **F2 depends on F1 being defined** (to know the LLM input/output shape). F1's output contract defines what `attach_citations.py` expects in `llm_output.json`. F1 does NOT need to be fully tested — just defined.
- **T007 depends on F3's `assets/data_dictionary_template.md`**. If F3 hasn't created this yet, T007 cannot produce output. T001–T006 work without F3.
- **T008 depends on F3's `assets/qa_report_template.md`**. Same constraint as T007.
- **T015 depends on F3's `assets/sample_schema.json`** for the 25-field test input. If F3 hasn't created this yet, use the 3-field test input from `specs/f2-scripts-data-dictionary/quickstart.md` and defer T015.
- **F3 depends on F2 being complete** for gold standard generation (SC-F3-006). F3 can build templates and sample schema without F2, but the gold standard examples require running the pipeline.

---

## FR Traceability

| Task(s) | FR | How It's Satisfied |
|---|---|---|
| T002 | **FR-012, FR-013** | `sanitize_for_markdown()`, `display_value_for_markdown()`, `safe_write_text()`, and template helpers directly enable formatted Markdown output |
| T003 | **FR-001** | Accepts JSON schema as input for data dictionary generation |
| T003 | **FR-005** | Input validation handled by deterministic script before LLM processing |
| T004 | **FR-006** | Extracts all fields from schema (field name, type, nullable, constraints/enums) |
| T005 | **FR-008** | Citations (source path, evidence refs) attached to every field |
| T005 | **FR-009** | Confidence scores validated and merged from LLM output |
| T006 | **FR-010** | `last_verified` timestamp added to every field |
| T007 | **FR-012** | Produces `data_dictionary.md` using F3 template |
| T008 | **FR-013** | Produces `qa_report.md` with coverage stats, confidence distribution, and flagged items |

---

## Notes

- [P] tasks = different files, no dependencies
- All Phase 2 and Phase 3 tasks write to different files, but they form a sequential pipeline — each script reads the previous script's output
- Phase 4 tasks are human-executed, not agent tasks
- `utils.py` (T002) is a shared helper module, not a pipeline script — it does not appear in the pipeline sequence and does not run independently
- Demo day deadline: May 7, 2026
- If the Cursor agent builds all scripts in one pass, Phases 2 and 3 effectively merge into a single build step — the phase separation exists for logical clarity and review
- Steps 1–6 (T003–T006) can be fully built and tested without F3. Steps 7–8 (T007–T008) require F3 templates
- Before starting a new pipeline run, clear `output/intermediate/` to prevent stale files from a previous run (see `specs/f2-scripts-data-dictionary/plan.md` Known Limitations)
- Total tasks: 18 (T001–T008 agent tasks, T009–T018 human validation tasks)
