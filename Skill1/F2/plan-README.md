# Implementation Plan: F2 — Scripts: Data Dictionary Generation

**Branch**: `f2-scripts-data-dictionary` | **Date**: 2026-04-10 | **Spec**: specs/f2-scripts-data-dictionary/spec.md

**Input**: Feature specification from `/specs/f2-scripts-data-dictionary/spec.md`

---

## Summary

F2 — Scripts: Data Dictionary Generation implements the deterministic Python pipeline that **controls the entire Data Dictionary skill's execution flow**. F2 owns everything that does NOT require an LLM: parsing, validating, extracting, merging, timestamping, assembling, and reporting. F2 also defines the input/output contracts with the LLM (F1) — controlling what metadata the LLM receives and validating what it returns.

The pipeline consists of **6 single-purpose Python scripts** that communicate via JSON files written to disk. Pre-LLM, the scripts validate the user's JSON schema file and extract field-level metadata into the format F1 expects. Post-LLM, they validate the LLM's output against the input (count check, field name matching), merge it with the extracted metadata and source citations, apply timestamps, assemble the final `data_dictionary.md`, and generate a `qa_report.md` with coverage statistics.

**Graceful degradation:** If the LLM (F1) goes offline or returns unusable output, F2's scripts still run and produce a data dictionary containing all raw metadata (field names, types, constraints, enums, source paths, timestamps) — but with placeholder values where LLM-generated content would normally appear. Placeholder values are:

- `description`: `"[LLM did not return a description]"`
- `confidence`: `"None"`
- `clarification_flag`: `true`
- `evidence_refs`: `["No evidence available — LLM did not process this field"]`
- `merge_status`: `"placeholder"`

The QA report notes that LLM-generated content is missing. This gives the team a structured starting point to manually write descriptions, or to re-run the skill when the LLM is back online. See data-model.md Section 6 for the full placeholder conventions table.

**Demo day target:** Process the UCI Credit Card dataset (25 fields) end-to-end, producing an audit-ready data dictionary and QA report by **May 7, 2026**. JSON input only. All fields processed in a single pass. `glossary_mapper.py` is documented in the feature spec as P3 but is **not included in this plan**.

### Pipeline Scripts Overview

| # | Script | What It Does | Why It Matters | When It Runs |
|---|---|---|---|---|
| 1 | `validate_input.py` | Checks that the user's file is valid JSON with the required structure (`table_name`, `source_file`, `fields`, `field_name` per field). If anything is wrong, it collects ALL errors and reports them at once. | **Gatekeeper.** Nothing else runs until the input is clean. Prevents garbage-in-garbage-out. | Pre-LLM — Step 1 |
| 2 | `extract_fields.py` | Reads the validated schema and pulls out every field's metadata (name, type, nullable, constraints, enums, comments). Builds a `source_path` tracing each field back to its origin file and table. Logs warnings for edge cases (e.g., duplicate field names) to `extraction_warnings.json`. | **Prepares the LLM's input.** The LLM can only write good descriptions if it receives clean, complete metadata. Also creates the traceability chain auditors need. | Pre-LLM — Step 2 |
| 3 | `attach_citations.py` | Takes what the LLM returned and checks it: Did it respond for every field? Do the field names match? Then merges the LLM's descriptions and scores with the original metadata. If the LLM missed fields or was offline, fills in placeholders. Each field gets a `merge_status` of `"matched"` (LLM responded) or `"placeholder"` (LLM failed/skipped). This field drives coverage calculations and the QA report's Processing Notes section. When `attach_citations.py` fixes an issue, it logs a correction object (`type`, `original`, `corrected`, `reason`) in the field's `corrections` array. Five correction types exist: `clarification_flag_override`, `name_case_mismatch`, `description_over_limit`, `order_mismatch`, `duplicate_name_in_output`. All corrections surface in the QA report's Merge Corrections section. See data-model.md for full definitions. | **Quality control on the LLM.** The LLM can make mistakes — skip fields, return wrong names, or go offline entirely. This script catches all of that, corrects what it can, logs every correction, and ensures no field is silently lost. | Post-LLM — Step 1 |
| 4 | `add_timestamps.py` | Adds a `last_verified` timestamp (same timestamp for every field in the run) to each field record. | **Audit trail.** Auditors need to know when each field was last verified. One timestamp per run keeps it consistent. | Post-LLM — Step 2 |
| 5 | `assemble_output.py` | Takes all the merged, timestamped field data and fills in the data dictionary template (from F3) to produce the final `data_dictionary.md`. Checks that the template file exists before proceeding — if missing, stops with a clear error message. | **Produces the deliverable.** This is the document the user actually receives — the whole point of the skill. | Post-LLM — Step 3 |
| 6 | `generate_qa_report.py` | Counts everything: how many fields got descriptions, how many have citations, how many are flagged for clarification. Produces `qa_report.md` with coverage stats and warnings. Checks that the template file exists before proceeding — if missing, stops with a clear error message. | **Trust but verify.** The QA report lets the user see at a glance whether the data dictionary is complete or if items need attention. | Post-LLM — Step 4 (final) |

### QA Report Structure

The QA report (`qa_report.md`) contains a Report Header plus 6 sections. Section names come from F3's QA report template (`qa_report_template.md`), which is the single source of truth for the QA report structure (FR-F3-005).

| Section | What It Contains | Always Present? |
|---------|-----------------|-----------------|
| Report Header | Table name, field count, run timestamp, pipeline version | Yes |
| 1. Coverage Statistics | How many fields got descriptions, percentage, placeholder count | Yes |
| 2. Confidence Distribution | Counts and percentages of High, Medium, Low, and N/A confidence fields | Yes |
| 3. Fields Requiring Clarification | List of fields where `clarification_flag` is `true` | Yes |
| 4. Merge Corrections | Table of corrections applied by `attach_citations.py` | Only if corrections exist |
| 5. Warnings | Warnings from `extract_fields.py` (e.g., duplicate field names), LLM output validation issues, missing LLM output | Only if warnings exist |
| 6. Processing Notes | LLM availability, fields processed, batching info, run timestamp | Yes |

### Pipeline Status (QA Report Processing Notes)

The Processing Notes section reports one of three statuses:

| Status | Meaning |
|--------|---------|
| `Complete` | All fields matched, no warnings |
| `Complete with warnings` | All fields matched, but extraction warnings or corrections exist |
| `Partial` | Some or all fields got placeholders (LLM failed partially or entirely) |

---

## Technical Context

**Language/Version**: Python 3.11+ (tested against whatever Python version Claude's sandbox provides)

**Primary Dependencies**: Python standard library only — **no third-party packages** (`pip install` is not guaranteed in Claude's sandbox). Key modules used:

- `json` — reading/writing all intermediate and output files (all 6 scripts use this)
- `datetime` — ISO 8601 timestamp generation (`add_timestamps.py` only)
- `os` / `sys` — file path handling, checking if files exist, creating folders, script execution (all scripts use this)

**Storage**: File-based. Scripts communicate via JSON files written to disk (see FR-F2-004). Final outputs are Markdown files (`data_dictionary.md`, `qa_report.md`). No database. All file paths are **relative** (not absolute) to avoid sandbox path issues.

**Testing**: Manual testing against UCI Credit Card dataset (25 fields). Success criteria SC-F2-001 through SC-F2-008 measured via:

- Automated: field count comparisons, source path presence checks, timestamp presence checks, determinism diffs (run 3x, compare outputs)
- Manual: validation accuracy (known-bad inputs), QA report cross-checks, graceful degradation simulation (remove `llm_output.json` and verify partial output)

**Target Platform**: Claude Agent Skills runtime — local execution within Claude's sandbox environment. No cloud deployment for demo day.

**Project Type**: Script pipeline (deterministic Python scripts within Claude Agent Skills architecture)

**Performance Goals**: Accuracy over speed (Constitution Principle 1). No latency SLA. All 25 fields processed in a single pipeline run. Deterministic — same input always produces same output (excluding timestamps).

**Constraints**:

- JSON input only for demo day (YAML/DDL is future scope)
- Only metadata sent to LLM — no raw PII, datasets, or credentials (Constitution Never-Ever Rules)
- File-based I/O only — no databases, no network calls, no microservices
- **Hard rule: Python standard library only** — no third-party packages. Any script that requires `pip install` is a plan violation.
- All fields processed in one pass — no chunking for demo day (25 fields is well within limits)
- Templates (`data_dictionary_template.md`, `qa_report_template.md`) are owned by F3 — F2 reads them but does not create or modify them
- **Template files must exist before the pipeline runs.** If a template is missing, `assemble_output.py` or `generate_qa_report.py` stops with a clear error message. This is a setup error, not a graceful degradation scenario.

**Scale/Scope**: Demo day: 25 fields, 1 table, 1 schema file. Chunking for 500+ field schemas is documented as future scope but not implemented.

---

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Constitution Principle | F2 Compliance | How F2 Follows This Rule |
|---|---|---|
| **Accuracy Over Speed** | ✅ Compliant | No speed requirements. Scripts prioritize correct extraction, correct validation, and correct counts over fast execution. `attach_citations.py` validates every LLM response before merging — never silently drops data. |
| **Graceful Degradation** | ✅ Compliant | If the LLM is offline or returns bad output, F2 still runs. `attach_citations.py` checks if `llm_output.json` exists — if not, it fills in placeholders and sets every field's `merge_status` to `"placeholder"`. The user gets a partial data dictionary with all raw metadata intact plus a QA report noting what's missing. **Note:** A missing F3 template is NOT a graceful degradation scenario — it's a setup error. See Constraints above. |
| **Simplicity First** | ✅ Compliant | 6 single-purpose Python scripts. Standard library only. File-based I/O. No databases, no frameworks, no cloud, no UI. Each script does one job and writes one file. |
| **Audit-Ready by Default** | ✅ Compliant | Every field includes a `source_path` citation tracing it back to the original schema file. `qa_report.md` is the first artifact an auditor reviews — it provides coverage stats, flags items needing attention, and shows at a glance whether the data dictionary is complete or has gaps. All corrections are logged in the field's `corrections` array — nothing is silently fixed. |
| **Never send raw PII to LLM** | ✅ Compliant | `extract_fields.py` is the **security boundary** — it decides exactly what metadata gets sent to the LLM (field names, types, constraints, enums, comments only). Actual data values from the dataset are never included. If this script has a bug, the security model breaks — so it must be carefully reviewed. |
| **Never send API keys or credentials** | ✅ Compliant | No credentials exist in any script or intermediate file. Scripts handle only schema metadata and generated documentation. |
| **Never send full source code or raw datasets** | ✅ Compliant | Only structured field metadata (extracted from the schema) is passed to the LLM. The raw schema file itself is not sent — only the extracted fields. |
| **Never commit secrets to the repo** | ✅ Compliant | Scripts contain no secrets. Intermediate JSON files contain only field metadata. |
| **Never opt into LLM provider training** | ✅ Compliant | No opt-in. This is a platform-level setting, not something F2 controls, but F2 does not do anything that would trigger or require opt-in. |

**No violations detected.** F2's design is fully aligned with the Constitution. This is consistent with F1's Constitution Check — both features pass with no violations, confirming the overall Data Dictionary skill design is aligned.

**Post-handoff note:** When Synchrony runs this skill on real internal data, field *names* in intermediate JSON files could be sensitive (e.g., `SSN`, `account_number`). Field names are metadata (not PII), but Synchrony's security team should review what field names are acceptable to process. This is Synchrony's responsibility per the Constitution's handoff boundary (Section 4).

---

## Project Structure

### Documentation (this feature)

    specs/f2-scripts-data-dictionary/
    ├── plan.md              # This file
    ├── research.md          # Phase 0 — Python stdlib patterns, JSON validation approaches, file-based pipeline design
    ├── data-model.md        # Phase 1 — intermediate JSON schemas, merged field structure, placeholder conventions
    ├── quickstart.md        # Phase 1 — how to run, test, and debug the 6-script pipeline
    ├── contracts.md         # Phase 1 — input/output agreements between F2 → F1 and F2 → F3
    └── tasks.md             # Phase 2 — implementation checklist (created separately)

### Source Code (repository root)

    skills/
    └── data-dictionary/
        └── scripts/                          # F2 deliverables — this plan's 6 Python scripts
            ├── validate_input.py             # Pre-LLM Step 1: validates user's JSON schema
            ├── extract_fields.py             # Pre-LLM Step 2: extracts field metadata for LLM (security boundary)
            ├── attach_citations.py           # Post-LLM Step 1: validates LLM output, merges with extracted metadata
            ├── add_timestamps.py             # Post-LLM Step 2: stamps every field with last_verified (ISO 8601)
            ├── assemble_output.py            # Post-LLM Step 3: fills F3 template → data_dictionary.md
            └── generate_qa_report.py         # Post-LLM Step 4: produces qa_report.md with coverage stats

### Runtime Output (created by scripts)

    skills/
    └── data-dictionary/
        └── output/                               # Created at runtime by os.makedirs()
            ├── data_dictionary.md                # Final deliverable (from assemble_output.py)
            ├── qa_report.md                      # Final deliverable (from generate_qa_report.py)
            └── intermediate/
                ├── validated_schema.json          # From validate_input.py
                ├── extracted_fields.json          # From extract_fields.py
                ├── extraction_warnings.json       # From extract_fields.py (conditional — only if warnings exist)
                ├── merged_fields.json             # From attach_citations.py
                └── timestamped_fields.json        # From add_timestamps.py

**Files F2 reads but does not create:**
- `user_input.json` — provided by the user via Claude upload, placed by SKILL.md (F1)
- `llm_output.json` — written by the LLM during the F1 step

**Structure Decision**: F2's deliverables are 6 Python scripts in `skills/data-dictionary/scripts/`.
This plan only shows what F2 owns — the scripts directory. Other components in the same skill folder
are owned by other features: `SKILL.md` (F1), `assets/` (F3). The `output/` directory and its
`intermediate/` subdirectory are created at runtime by F2 scripts via `os.makedirs()` — they are
not checked into the repository. This keeps the Source Code section focused on F2's actual
deliverables (Constitution Principle 3: Simplicity First) and is consistent with how F1 and F3
plans show only their own files.

### Who Owns What

| Folder/File | Owner | F2's Relationship |
|---|---|---|
| `SKILL.md` | F1 | F2 doesn't touch this. The SKILL.md tells Claude when to call each script. |
| `scripts/` | **F2 (this plan)** | F2 creates and maintains all 6 scripts. This is the deliverable. |
| `assets/` | F3 | F2 reads the templates and test data but does NOT create or modify them. These are **pending F3 delivery** (Open Items #1, #2, #3 in the feature spec). |
| `output/` | Shared | Final deliverables are produced by F2 scripts. `llm_output.json` is written by F1. |
| `output/intermediate/` | F2 | All intermediate JSON files are created by F2 scripts (except `llm_output.json` which F1 writes). |

### Implementation Notes

- **Folder creation:** Scripts must create `output/` and `output/intermediate/` if they don't already exist (using `os.makedirs()`). Don't assume the folders are pre-created.
- **File paths:** Scripts use relative paths with the `intermediate/` prefix for handoff files. For example, `attach_citations.py` reads from `output/intermediate/extracted_fields.json`, not just `extracted_fields.json`. Final deliverables write directly to `output/`.
- **Template existence check:** `assemble_output.py` and `generate_qa_report.py` must verify their F3 template file exists before attempting to read it. If missing, print an error message with the exact expected path and stop. Do not attempt to produce output without the template.
- **`glossary_mapper.py`** is P3 — not created, not tested, not part of this project structure for demo day.

### What F2 Does NOT Create

- `SKILL.md` — F1's deliverable
- `data_dictionary_template.md` — F3's deliverable
- `qa_report_template.md` — F3's deliverable
- `sample_schema.json` — F3's deliverable

### F3 Dependency Risk

⚠️ Two of F2's scripts depend on F3 templates that don't exist yet:

| F2 Script | Needs From F3 | Open Item # |
|---|---|---|
| `assemble_output.py` | `data_dictionary_template.md` | Open Item #1 |
| `generate_qa_report.py` | `qa_report_template.md` | Open Item #2 |
| All scripts (for testing) | `sample_schema.json` | Open Item #3 |

**Risk:** If F3 delivers templates late, F2 cannot fully test `assemble_output.py` or `generate_qa_report.py`. **Mitigation:** F2 team can create simple placeholder templates for early testing, then swap in F3's final templates when ready. Coordinate with F3 early.

**Missing template behavior:** If a template file is not found at the expected path, the script stops immediately with a clear error message. This is a setup error, not a runtime failure — see contracts.md (Contract 3: F2 → F3) for the full agreement.

**Structure Decision:** Single skill folder with subfolders for scripts, assets, and output. Chosen because it keeps the entire Data Dictionary skill self-contained, matches the Claude Agent Skills architecture, and aligns with the structure F1's plan proposed.

---

## Complexity Tracking

No Constitution violations detected. No complexity justifications required.
