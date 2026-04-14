---
description: "Task list for F3 — Assets: Data Dictionary Generation"
---

# Tasks: F3 — Assets: Data Dictionary Generation

**Input**: Design documents from `/specs/f3-assets-data-dictionary/`
**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, contracts.md, quickstart.md

**Tests**: Testing and validation tasks are included and marked as human-executed tasks (not agent tasks). F3 builds static asset files; correctness is validated through manual review and cross-feature integration checks.

**Organization**: All implementation tasks belong to User Story 1 (US1 — Build Data Dictionary Assets, P1). Validation tasks are grouped in the Polish phase. F3 deliverables are grouped by asset role: templates first, then sample schema, then gold standards, then the deferred glossary placeholder.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1 for all F3 tasks)
- **[HUMAN]**: Task executed by the developer after the agent finishes — not an agent task
- Include exact file paths in descriptions

## Path Conventions

- **SKILL root** lives at `.cursor/skills/data_dictionary/`
- **SKILL.md** lives at `.cursor/skills/data_dictionary/SKILL.md`
- **Scripts** live at `.cursor/skills/data_dictionary/scripts/`
- **Assets** live at `.cursor/skills/data_dictionary/assets/`
- **Runtime output** lives at `.cursor/skills/data_dictionary/output/`
- **Intermediate output** lives at `.cursor/skills/data_dictionary/output/intermediate/`
- **Spec documents** live at `specs/f3-assets-data-dictionary/`
- **Project-level docs** live in `specs/` (constitution.md, project-spec.md, data-dictionary-master.md)

**Agent Workspace**: Before execution, copy all F3 planning documents and project-level documents into `.cursor/skills/data_dictionary/docs/`. The agent reads from `docs/` and builds asset files into `.cursor/skills/data_dictionary/assets/`. The `docs/` folder is a working folder — it is not part of the canonical repository structure and should not be checked in. If the `docs/` folder already exists from F1/F2, add the F3 documents to it.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Create the asset directory and empty asset files so the agent has targets to write to.

- [ ] T001 [US1] Create directory `.cursor/skills/data_dictionary/assets/` under the skill root. This directory holds all F3 static files and must sit alongside `.cursor/skills/data_dictionary/SKILL.md` and `.cursor/skills/data_dictionary/scripts/` so the skill remains self-contained in Cursor.

- [ ] T002 [US1] Create empty file `.cursor/skills/data_dictionary/assets/data_dictionary_template.md` with a top-level heading `# Data Dictionary — {table_name}` and a placeholder line `<!-- F2 reads this template structure at runtime -->`.

- [ ] T003 [US1] Create empty file `.cursor/skills/data_dictionary/assets/qa_report_template.md` with a top-level heading `# QA Report — {table_name}` and a placeholder line `<!-- F2 reads this template structure at runtime -->`.

- [ ] T004 [US1] Create empty file `.cursor/skills/data_dictionary/assets/sample_schema.json` with the three required top-level keys stubbed: `"table_name"`, `"source_file"`, and `"fields": []`.

- [ ] T005 [US1] Create empty file `.cursor/skills/data_dictionary/assets/example_data_dictionary.md` with a top-level heading `# Data Dictionary — credit_card_clients` and a placeholder line `<!-- Gold standard output for sample_schema.json -->`.

- [ ] T006 [US1] Create empty file `.cursor/skills/data_dictionary/assets/example_qa_report.md` with a top-level heading `# QA Report — credit_card_clients` and a placeholder line `<!-- Gold standard QA report for sample_schema.json -->`.

- [ ] T007 [US1] Create empty file `.cursor/skills/data_dictionary/assets/glossary_template.json` with the top-level key `"glossary": []` as a deferred placeholder for the P3 glossary feature.

**Checkpoint**: Asset directory exists and all 6 F3 files exist at the correct Cursor-relative paths. F2 can now reference these files by path.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Define the structural contracts that all F3 assets must follow before filling in content. These structures are authoritative for F2 runtime behavior and gold standard validation.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete. The templates and schema contract must be locked before the gold standards are written, because the gold standards must follow the templates exactly.

- [ ] T008 [US1] Write the **data dictionary template header and section skeleton** in `.cursor/skills/data_dictionary/assets/data_dictionary_template.md`. Include: (1) document heading with `{table_name}` placeholder; (2) metadata lines for `**Source File:** {source_file}` and `**Generated:** {generation_date}`; (3) a horizontal rule; (4) `## Field Summary`; (5) `## Evidence & Citations`; (6) `## Confidence Legend`; (7) the static footer note for `[NEEDS CLARIFICATION]`. Reference: `specs/f3-assets-data-dictionary/data-model.md` Section 1, `specs/f3-assets-data-dictionary/contracts.md` Contract 3.

- [ ] T009 [US1] Write the **main table header** in `.cursor/skills/data_dictionary/assets/data_dictionary_template.md` with the exact 8 columns in this exact order: `field_name`, `type`, `nullable`, `constraints`, `enums`, `description`, `confidence`, `last_verified`. Include the Markdown separator row directly beneath the header. State no extra columns and no glossary column for demo day. Reference: `specs/f3-assets-data-dictionary/data-model.md` Section 1 (Main Table Section), `specs/f3-assets-data-dictionary/research.md` D-007.

- [ ] T010 [US1] Write the **Evidence & Citations entry format** in `.cursor/skills/data_dictionary/assets/data_dictionary_template.md` so F2 can render one entry per field using: `### FIELD_NAME`, `**Source:** sample_schema.json → table: credit_card_clients → field: FIELD_NAME`, and a numbered `**Evidence:**` list with one item per line. Do not use semicolon-delimited evidence in this section. Reference: `specs/f3-assets-data-dictionary/data-model.md` Section 1 (Evidence & Citations), `specs/f3-assets-data-dictionary/contracts.md` Contract 3.

- [ ] T011 [US1] Write the **confidence legend and generation notes** in `.cursor/skills/data_dictionary/assets/data_dictionary_template.md`. Include the exact High / Medium / Low explanations, the note that descriptions are AI-generated drafts for human review, and the note that fields marked `[NEEDS CLARIFICATION]` require human attention before production use. Reference: `specs/f3-assets-data-dictionary/data-model.md` Section 1 (Footer Section).

- [ ] T012 [US1] Write the **QA report header and section skeleton** in `.cursor/skills/data_dictionary/assets/qa_report_template.md`. Include metadata lines for run timestamp, pipeline version, input file, and table name, followed by the exact 6 section headings in order: `## 1. Coverage Statistics`, `## 2. Confidence Distribution`, `## 3. Fields Requiring Clarification`, `## 4. Merge Corrections`, `## 5. Warnings`, `## 6. Processing Notes`. Reference: `specs/f3-assets-data-dictionary/plan.md` QA Report Structure, `specs/f3-assets-data-dictionary/contracts.md` Contract 2.

- [ ] T013 [US1] Write the **placeholder and substitution rules** into both template files as authoring constraints: only `{table_name}`, `{source_file}`, and `{generation_date}` may appear as live placeholders; table rows and evidence entries are structure-based, not field-by-field placeholders; no extra placeholder literals may appear in static footer text because F2 uses global `.replace()`. Reference: `specs/f3-assets-data-dictionary/contracts.md` Contract 3 (Placeholder Rules).

- [ ] T014 [US1] Write the **Markdown formatting constraints** into the template authoring notes for both `.cursor/skills/data_dictionary/assets/data_dictionary_template.md` and `.cursor/skills/data_dictionary/assets/qa_report_template.md`: no pipe characters in table cell content, no newlines inside table cells, always include the header separator row, avoid angle brackets in content, and keep tables machine-readable by F2. Reference: `specs/f3-assets-data-dictionary/contracts.md` Contract 3, `specs/f3-assets-data-dictionary/quickstart.md` Troubleshooting.

- [ ] T015 [US1] Define the **sample schema contract** in `.cursor/skills/data_dictionary/assets/sample_schema.json` by locking the top-level keys (`table_name`, `source_file`, `fields`) and per-field keys (`field_name` required; `type`, `nullable`, `constraints`, `enums`, `schema_comments` optional). Preserve JSON-only demo scope. Reference: `specs/f3-assets-data-dictionary/data-model.md` Section 3, `specs/f3-assets-data-dictionary/contracts.md` Cross-Reference: Alignment with F1 and F2 Contracts.

**Checkpoint**: Both templates have authoritative structure, placeholder rules are locked, and the sample schema contract is defined. Gold standard writing can now begin.

---

## Phase 3: User Story 1 - Build Data Dictionary Assets (Priority: P1) 🎯 MVP

**Goal**: Create the full F3 asset package — two runtime templates, one 25-field sample schema, two gold standard outputs, and one deferred glossary placeholder — so Skill 1 has its output contract, test harness, and validation anchor.

**Independent Test**: Open the completed assets side by side with the F3 data model and contracts. Verify the templates are structurally correct, the sample schema is valid JSON with 25 fields, and the gold standards follow the template structure exactly and are internally consistent.

### Implementation for User Story 1

- [ ] T016 [US1] Complete `.cursor/skills/data_dictionary/assets/data_dictionary_template.md` as the final runtime template. Ensure the file contains: header metadata placeholders, the exact 8-column table header, an empty structure for the Evidence & Citations section, the confidence legend, the generation note, and the `[NEEDS CLARIFICATION]` note. Do not include field-specific rows in the template — F2 generates rows dynamically. Reference: `specs/f3-assets-data-dictionary/research.md` D-001, D-002; `specs/f3-assets-data-dictionary/data-model.md` Section 1.

- [ ] T017 [US1] Complete `.cursor/skills/data_dictionary/assets/qa_report_template.md` as the final runtime template. Include the report header, the 6 authoritative sections in order, and formatting that cleanly supports: tables for Coverage Statistics and Confidence Distribution; a field list for Fields Requiring Clarification; a correction table or fallback line for Merge Corrections; a warnings block or fallback line for Warnings; and a notes table for Processing Notes. Reference: `specs/f3-assets-data-dictionary/plan.md` QA Report Structure, `specs/f3-assets-data-dictionary/data-model.md` Section 2.

- [ ] T018 [US1] Populate `.cursor/skills/data_dictionary/assets/sample_schema.json` with the full 25 fields from the UCI Credit Card dataset in the exact field order documented in the data model: `ID`, `LIMIT_BAL`, `SEX`, `EDUCATION`, `MARRIAGE`, `AGE`, `PAY_0`, `PAY_2`, `PAY_3`, `PAY_4`, `PAY_5`, `PAY_6`, `BILL_AMT1`, `BILL_AMT2`, `BILL_AMT3`, `BILL_AMT4`, `BILL_AMT5`, `BILL_AMT6`, `PAY_AMT1`, `PAY_AMT2`, `PAY_AMT3`, `PAY_AMT4`, `PAY_AMT5`, `PAY_AMT6`, `default.payment.next.month`. Set `"table_name": "credit_card_clients"` and `"source_file": "sample_schema.json"`. Reference: `specs/f3-assets-data-dictionary/data-model.md` Section 3.

- [ ] T019 [US1] Ensure `.cursor/skills/data_dictionary/assets/sample_schema.json` includes the required metadata variety for testing all paths: at least 3 fields with missing optional metadata, at least 1 field with cryptic numeric enums, at least 1 field with constraints, at least 1 oddly formatted field name with dots, and at least one numbered field series. Use intentional metadata gaps rather than making all fields fully documented. Reference: `specs/f3-assets-data-dictionary/data-model.md` Section 3 (Required Variety), `specs/f3-assets-data-dictionary/research.md` Section 5.

- [ ] T020 [US1] Create `.cursor/skills/data_dictionary/assets/example_data_dictionary.md` as the gold standard output for `sample_schema.json`. Follow the final `data_dictionary_template.md` structure exactly: same header layout, same 8-column table order, same Evidence & Citations structure, same footer. Include all 25 fields, one table row per field, and one evidence entry per field. Reference: `specs/f3-assets-data-dictionary/data-model.md` Section 4, `specs/f3-assets-data-dictionary/contracts.md` Contract 4.

- [ ] T021 [US1] Write all 25 **gold standard descriptions** in `.cursor/skills/data_dictionary/assets/example_data_dictionary.md` in plain language, each at 25 words or fewer. Keep them accurate to metadata only, not actual data values. Do not invent business logic or unstated semantics. Reference: `specs/project-spec.md` FR-007, `specs/f3-assets-data-dictionary/data-model.md` Section 4, `.cursor/skills/data_dictionary/SKILL.md` Description rules.

- [ ] T022 [US1] Assign **gold standard confidence scores** in `.cursor/skills/data_dictionary/assets/example_data_dictionary.md` using the F1 rubric from `.cursor/skills/data_dictionary/SKILL.md`. Ensure the completed gold standard contains at least 1 `"High"`, 1 `"Medium"`, and 1 `"Low"` confidence field, and use the rubric consistently across related field series. Reference: `specs/f3-assets-data-dictionary/data-model.md` Section 4 (confidence distribution coverage), `specs/f3-assets-data-dictionary/research.md` Expected Confidence Distribution, `.cursor/skills/data_dictionary/SKILL.md`.

- [ ] T023 [US1] Write **gold standard evidence reasoning** for all 25 fields in the Evidence & Citations section of `.cursor/skills/data_dictionary/assets/example_data_dictionary.md`. Each field's evidence must cite only contracted signals (`field_name`, `type`, `nullable`, `constraints`, `enums`, `source_path`, `schema_comments`) and explain why the confidence score was assigned. Do not use unsupported reasoning. Reference: `.cursor/skills/data_dictionary/SKILL.md` evidence_refs rules, `specs/f3-assets-data-dictionary/contracts.md` Contract 4.

- [ ] T024 [US1] Apply the **clarification rendering rules** in `.cursor/skills/data_dictionary/assets/example_data_dictionary.md`: every Low confidence field must have `[NEEDS CLARIFICATION]` prepended to its description in the main table, and the same fields must read as clarification-required in the gold standard interpretation. Do not create a separate clarification column. Reference: `specs/f3-assets-data-dictionary/data-model.md` Section 1 (`[NEEDS CLARIFICATION]` rendering), `.cursor/skills/data_dictionary/SKILL.md`.

- [ ] T025 [US1] Create `.cursor/skills/data_dictionary/assets/example_qa_report.md` as the gold standard QA output derived from `example_data_dictionary.md`. Follow the final `qa_report_template.md` structure exactly, with the same header layout and same 6 sections in order. Reference: `specs/f3-assets-data-dictionary/data-model.md` Section 5, `specs/f3-assets-data-dictionary/contracts.md` Contract 4.

- [ ] T026 [US1] Populate the **Coverage Statistics** and **Confidence Distribution** sections in `.cursor/skills/data_dictionary/assets/example_qa_report.md` directly from counts in `example_data_dictionary.md`. Set Total Fields = 25, Fields with Descriptions = 25, Fields with Citations = 25, Fields with Confidence Scores = 25, Completeness = 100%, and compute the exact High / Medium / Low / N/A counts and percentages from the gold standard. Reference: `specs/f3-assets-data-dictionary/data-model.md` Section 5, `specs/data-dictionary-master.md` QA example.

- [ ] T027 [US1] Populate the **Fields Requiring Clarification** section in `.cursor/skills/data_dictionary/assets/example_qa_report.md` with every field whose gold standard description in `example_data_dictionary.md` is marked `[NEEDS CLARIFICATION]`. Include field name, current description, confidence, and a concise reason grounded in the field's evidence. Reference: `specs/f3-assets-data-dictionary/data-model.md` Section 5, `specs/f3-assets-data-dictionary/contracts.md` Contract 4.

- [ ] T028 [US1] Populate the **Merge Corrections**, **Warnings**, and **Processing Notes** sections in `.cursor/skills/data_dictionary/assets/example_qa_report.md` for the gold standard clean-run scenario. Use `No merge corrections.` and `No warnings.` where appropriate, and set processing notes to reflect demo-day assumptions: LLM available, all fields processed in a single pass, run timestamp example present, and pipeline status complete. Reference: `specs/f3-assets-data-dictionary/data-model.md` Section 5, `specs/data-dictionary-master.md`.

- [ ] T029 [P] [US1] Populate `.cursor/skills/data_dictionary/assets/glossary_template.json` with the deferred P3 placeholder structure: top-level `"glossary"` array plus at least two example `{ "term": ..., "label": ... }` entries such as `LIMIT_BAL` and `PAY_0`. Keep the file clearly marked as not used for demo day. Reference: `specs/f3-assets-data-dictionary/data-model.md` Section 6.

**Checkpoint**: All six F3 asset files are complete. Templates are ready for F2 runtime use, the sample schema is ready for end-to-end testing, and the gold standards are ready for comparison and SKILL.md anchoring.

---

## Phase 4: User Story 2 - Validate Asset Quality and Cross-File Consistency (Priority: P2)

**Goal**: Prove the completed F3 assets are structurally correct, internally consistent, aligned with the SKILL rubric, and safe for F2 to consume.

**Independent Test**: Starting from the completed files alone, a reviewer should be able to verify structure, counts, field order, and alignment without running code.

### Implementation for User Story 2

- [ ] T030 [HUMAN] [US2] Run **SC-F3-001 template validation** on `.cursor/skills/data_dictionary/assets/data_dictionary_template.md` and `.cursor/skills/data_dictionary/assets/qa_report_template.md`. Verify: all 8 columns are present in the data dictionary template, columns are in the correct order, all 6 QA report sections are present and correctly named, the 3 header placeholders are present, footer notes exist, and both files render as valid Markdown. Reference: `specs/f3-assets-data-dictionary/data-model.md` Section 7 (Template Validation).

- [ ] T031 [HUMAN] [US2] Run **SC-F3-003 sample schema validation** on `.cursor/skills/data_dictionary/assets/sample_schema.json`. Verify: valid JSON, required top-level keys present, 25 fields present, every field has `field_name`, at least 3 fields have missing optional metadata, at least 1 field has cryptic enums, and at least 1 field has constraints. Reference: `specs/f3-assets-data-dictionary/data-model.md` Section 7 (Sample Schema Validation).

- [ ] T032 [HUMAN] [US2] Run **SC-F3-002 structural gold standard validation** on `.cursor/skills/data_dictionary/assets/example_data_dictionary.md`. Verify: structure matches `data_dictionary_template.md` exactly, all 25 fields appear in the main table, all 25 evidence entries appear below, evidence entry order matches table row order, and footer matches the template. Reference: `specs/f3-assets-data-dictionary/data-model.md` Section 7 (Gold Standard Validation).

- [ ] T033 [HUMAN] [US2] Run **SC-F3-004 content accuracy validation** on `.cursor/skills/data_dictionary/assets/example_data_dictionary.md`. Review every description against UCI/Kaggle field meaning, verify each description is 25 words or fewer, verify confidence scores follow the rubric in `.cursor/skills/data_dictionary/SKILL.md`, verify Low confidence fields are marked for clarification, and verify evidence reasoning is realistic and not tautological. Reference: `specs/f3-assets-data-dictionary/plan.md` Technical Context (Testing), `specs/f3-assets-data-dictionary/research.md` Section 4, `.cursor/skills/data_dictionary/SKILL.md`.

- [ ] T034 [HUMAN] [US2] Run **SC-F3-005 QA consistency validation** on `.cursor/skills/data_dictionary/assets/example_qa_report.md` against `.cursor/skills/data_dictionary/assets/example_data_dictionary.md`. Verify every metric by counting directly from the gold standard data dictionary, verify High + Medium + Low + N/A = 25, and verify every field marked `[NEEDS CLARIFICATION]` in the data dictionary appears in the QA clarification section. Reference: `specs/f3-assets-data-dictionary/data-model.md` Section 5 and Section 7.

- [ ] T035 [HUMAN] [US2] Run a **field-name and field-order consistency audit** across `.cursor/skills/data_dictionary/assets/sample_schema.json`, `.cursor/skills/data_dictionary/assets/example_data_dictionary.md`, and `.cursor/skills/data_dictionary/assets/example_qa_report.md`. Verify the 25 field names match exactly, the order is preserved between the sample schema and the gold standard data dictionary, and there are no typos such as `PAY_1` replacing `PAY_0`. Reference: `specs/f3-assets-data-dictionary/quickstart.md` Troubleshooting (Cross-File Consistency Problems).

- [ ] T036 [HUMAN] [US2] Run a **template placeholder audit** on `.cursor/skills/data_dictionary/assets/data_dictionary_template.md` and `.cursor/skills/data_dictionary/assets/qa_report_template.md`. Verify there are no placeholder literals beyond `{table_name}`, `{source_file}`, and `{generation_date}`, and verify these literals do not appear accidentally inside static explanatory text. Reference: `specs/f3-assets-data-dictionary/contracts.md` Contract 3.

**Checkpoint**: F3 assets have passed all manual quality gates that do not require F2 execution. They are now ready for integration-aware review.

---

## Phase 5: User Story 3 - Integration Readiness and Maintenance Workflow (Priority: P3)

**Goal**: Confirm F3 is ready for F2 runtime integration and future maintenance without breaking the gold standard / QA / SKILL.md dependency chain.

**Independent Test**: A developer unfamiliar with F3 should be able to follow the quickstart and update process without introducing structural inconsistencies.

### Implementation for User Story 3

- [ ] T037 [HUMAN] [US3] Run **SC-F3-006 integration readiness review** against F2 expectations. Verify that `.cursor/skills/data_dictionary/assets/data_dictionary_template.md`, `.cursor/skills/data_dictionary/assets/qa_report_template.md`, and `.cursor/skills/data_dictionary/assets/sample_schema.json` exist at the exact paths F2 should read from within the Cursor skill package. Confirm the F2 scripts or their path configuration will reference `.cursor/skills/data_dictionary/assets/` rather than a repo-root `assets/` directory. Reference: `specs/f3-assets-data-dictionary/contracts.md` Contract 2, `specs/f2-scripts-data-dictionary/tasks.md`.

- [ ] T038 [HUMAN] [US3] Run the **gold standard update-chain review** using the documented 5-step process. Starting from `.cursor/skills/data_dictionary/assets/example_data_dictionary.md`, verify the team can answer: (1) when to update `.cursor/skills/data_dictionary/assets/example_qa_report.md`; (2) when to update `.cursor/skills/data_dictionary/SKILL.md`; (3) how to re-check the confidence totals; and (4) how to obtain team sign-off. Reference: `specs/f3-assets-data-dictionary/plan.md` Gold Standard Update Process, `specs/f3-assets-data-dictionary/quickstart.md` How to Update the Gold Standard.

- [ ] T039 [HUMAN] [US3] Validate **SKILL.md alignment** by checking that the gold standard contains at least one candidate High, one Medium, and one Low confidence example suitable for worked-example anchoring in `.cursor/skills/data_dictionary/SKILL.md`. If the fields currently used in SKILL.md differ from the final gold standard wording, document that the SKILL.md examples must be updated manually. Reference: `specs/f3-assets-data-dictionary/contracts.md` Contract 1, `.cursor/skills/data_dictionary/SKILL.md`.

- [ ] T040 [HUMAN] [US3] Run **quickstart.md validation** against the final F3 assets. Walk through `specs/f3-assets-data-dictionary/quickstart.md` as if you were a new team member and verify the instructions correctly describe the file locations, usage, update chain, troubleshooting, and what can be done before F2 is fully wired. Flag any mismatches between the quickstart and the final asset set. Reference: `specs/f3-assets-data-dictionary/quickstart.md`.

- [ ] T041 [HUMAN] [US3] Run a **Constitution compliance review** across all files in `.cursor/skills/data_dictionary/assets/`. Verify there is no raw PII, no secrets, no credentials, no real dataset values, only metadata-level examples, UTF-8-safe text content, and no scope creep into out-of-scope governance or deployment features. Reference: `specs/constitution.md` Section 3 (Never-Ever Rules), `specs/f3-assets-data-dictionary/plan.md` Constitution Check.

**Checkpoint**: F3 assets are maintainable, integration-ready, and aligned with both F1 and F2.

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Final review items that affect multiple F3 assets and the broader Skill 1 workflow.

- [ ] T042 [HUMAN] [P] [US1] Documentation cleanup pass across `.cursor/skills/data_dictionary/assets/` to ensure headings, spacing, Markdown table alignment, and wording are consistent across both templates and both gold standards.

- [ ] T043 [HUMAN] [US1] Edge-case robustness review across all F3 Markdown files. Verify there are no unescaped pipe characters, no accidental line breaks inside table cells, no missing separator rows, and no angle-bracket content that would disappear in Markdown rendering. Reference: `specs/f3-assets-data-dictionary/data-model.md` Section 8, `specs/f3-assets-data-dictionary/quickstart.md` Troubleshooting.

- [ ] T044 [HUMAN] [US1] Future-schema readiness review. Confirm the templates in `.cursor/skills/data_dictionary/assets/data_dictionary_template.md` and `.cursor/skills/data_dictionary/assets/qa_report_template.md` are structure-based, not 25-field-specific placeholder sheets, so they can support post-handoff schemas with different field counts. Reference: `specs/f3-assets-data-dictionary/research.md` D-001.

- [ ] T045 [HUMAN] [US1] Final sign-off review. Confirm all 6 asset files exist, all 5 pre-F2 success criteria have passed, all required file paths align with the Cursor skill structure, and any remaining F2-dependent validation is clearly noted as integration follow-up rather than unresolved ambiguity.

---

## Dependencies & Execution Order

- **Setup (Phase 1)**: No dependencies — can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion — BLOCKS all user stories
- **User Stories (Phase 3+)**: All depend on Foundational phase completion
  - User stories can then proceed in parallel where staffing allows, though F3 is small enough that sequential execution is usually clearer
  - Or sequentially in priority order (P1 → P2 → P3)
- **Polish (Phase 6)**: Depends on all desired user stories being complete

### User Story Dependencies

- **User Story 1 (P1)**: Can start after Foundational (Phase 2) — no dependencies on other stories
- **User Story 2 (P2)**: Can start after User Story 1 content exists, because validation requires completed templates, schema, and gold standards
- **User Story 3 (P3)**: Can start after User Story 2, because integration readiness depends on validated assets and consistent update workflow

### Within Each User Story

- User Story 1 should proceed in this order:
  - Templates first (T016–T017)
  - Sample schema second (T018–T019)
  - Gold standard data dictionary third (T020–T024)
  - Gold standard QA report fourth (T025–T028)
  - Glossary placeholder last (T029)
- User Story 2 validation tasks are human-executed and should run after User Story 1 is complete
- User Story 3 maintenance and integration tasks are human-executed and should run after User Story 2 validation passes

### Parallel Opportunities

- In Phase 1, T002–T007 can run in parallel after T001 creates the assets directory
- In Phase 2, T008–T015 write to overlapping files, so keep them sequential for clarity and to avoid conflicts
- In Phase 3, T016 and T017 can be developed in parallel because they write to different template files
- In Phase 3, T018 and T019 are sequential because the schema must exist before its variety can be audited
- In Phase 3, T025–T028 depend on the completed gold standard data dictionary, so they are sequential after T020–T024
- In Phases 4–6, many human validation tasks can be done in parallel by different reviewers once the files are complete

---

## Parallel Example: User Story 1

```bash
# Launch the two template tasks together:
Task: "Complete .cursor/skills/data_dictionary/assets/data_dictionary_template.md as the final runtime template"
Task: "Complete .cursor/skills/data_dictionary/assets/qa_report_template.md as the final runtime template"

# After the gold standard data dictionary is complete, launch QA and glossary work separately:
Task: "Create .cursor/skills/data_dictionary/assets/example_qa_report.md as the gold standard QA output"
Task: "Populate .cursor/skills/data_dictionary/assets/glossary_template.json with the deferred P3 placeholder structure"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational (CRITICAL - blocks all stories)
3. Complete Phase 3: User Story 1
4. **STOP and VALIDATE**: Review the templates, schema, and gold standards independently
5. Hand off to F2 integration work if ready

### Incremental Delivery

1. Complete Setup + Foundational → F3 structure ready
2. Add User Story 1 → Review independently → F3 MVP complete
3. Add User Story 2 → Validate independently → F3 quality gate complete
4. Add User Story 3 → Validate independently → F3 maintenance/integration readiness complete
5. Each story adds value without breaking previous F3 deliverables

### Parallel Team Strategy

With multiple developers:

1. Team completes Setup + Foundational together
2. Once Foundational is done:
   - Developer A: data dictionary template + sample schema
   - Developer B: QA report template + gold standard QA report
   - Developer C: gold standard data dictionary + consistency review
3. Team reconverges for cross-file validation and final sign-off

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- Each user story should be independently completable and testable
- F3 is static, but it is not lightweight in importance — the templates are runtime contracts for F2 and the gold standards are the answer keys for Skill 1
- Validation is not optional quality theater here; it is part of the feature because F3 has no executable code of its own
- Commit after each task or logical group
- Stop at checkpoints to validate independently
- Avoid vague tasks, hidden assumptions about file paths, or any path references to a repo-root `assets/` directory that conflict with the actual Cursor skill structure
