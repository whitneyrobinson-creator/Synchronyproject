---
description: "Task list for F4 — RCSA Assets (Templates, Sample Data, and Control Library)"
---

# Tasks: F4 — RCSA Assets (Templates, Sample Data, and Control Library)

**Input**: Design documents from `/specs/f4-rcsa-assets/`
**Prerequisites**: plan.md (required), spec.md (required for user stories), constitution-v0.1.md

**Functional Requirements**: FR-021 (RCSA templates and citation format), FR-022 (sample data and control library)

**Deliverables**: 6 static asset files in `.cursor/skills/rcsa/assets/` — no runtime code, no external dependencies

**Tests**: No automated test suite. Verification is handled in Phase 6 through manual acceptance checks (coverage verification, security review, encoding validation).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which functional requirement this task belongs to (FR-021 or FR-022)
- Exact file paths included in all task descriptions

## Path Conventions

All files reside under `.cursor/skills/rcsa/assets/` with two subdirectories:

```
.cursor/skills/rcsa/assets/
├── sample-data/
│   ├── sample_control_library.json
│   ├── sample_artifact_index.json
│   └── sample_test_catalog.json
└── templates/
    ├── citation_format.md
    ├── rcsa_control_narratives_template.md
    └── validation_report_template.md
```

---

## Phase 1: Setup

**Purpose**: Create the directory structure for F4 assets. If directories already exist from prior features, this task is a no-op — verify and move on.

- [ ] T001 Create directory structure at `.cursor/skills/rcsa/assets/` with `sample-data/` and `templates/` subdirectories

**Acceptance Criteria for T001**:
- `.cursor/skills/rcsa/assets/sample-data/` exists
- `.cursor/skills/rcsa/assets/templates/` exists
- No extra directories or files created

**Checkpoint**: Directory structure exists and matches the path conventions above. *(Satisfies AS-11 partially — directory structure ready)*

---

## Phase 2: Foundational (Blocking Prerequisite)

**Purpose**: Create the control library — the single file that every other F4 deliverable references. This defines the 4 controls, their evidence types, and the JSON schema pattern used across all sample data files.

**⚠️ CRITICAL**: No other F4 task can begin until this phase is complete. The control library establishes the control IDs, evidence type names, and `_meta` schema pattern that all downstream files depend on.

- [ ] T002 [FR-022] Create `sample_control_library.json` at `.cursor/skills/rcsa/assets/sample-data/sample_control_library.json`

**Acceptance Criteria for T002**:
- File parses with `json.load()` without error
- Contains a `_meta` object with `"scale": "demo"`
- Contains exactly 4 controls with IDs: `AC` (Access Control), `CM` (Change Management), `DQ` (Data Quality), `IH` (Incident Handling)
- Each control has the fields: `id`, `name`, `objective`, and `evidence_types`
- Each control has ≥2 evidence types (11 total across all 4 controls)
- Evidence type entries include enough detail for downstream files to reference them (at minimum: a type identifier and a human-readable description)
- All file paths are relative — no absolute paths
- Zero PII, secrets, or real Synchrony identifiers
- File is UTF-8 encoded

**Checkpoint**: Control library is valid JSON, contains all 4 controls with 11 evidence types, and establishes the `_meta` schema pattern. *(Satisfies AS-01, AS-02)*

---

## Phase 3: Sample Data

**Purpose**: Create the artifact index and test catalog. Both reference the control IDs and evidence types defined in the control library (T002). These two files are independent of each other and can be built in parallel.

- [ ] T003 [P] [FR-022] Create `sample_artifact_index.json` at `.cursor/skills/rcsa/assets/sample-data/sample_artifact_index.json`

**Acceptance Criteria for T003**:
- File parses with `json.load()` without error
- Contains a `_meta` object with `"scale": "demo"`
- Contains 10–20 artifact entries
- Each artifact has at minimum: a file path (relative), a description, and a mapping to one or more control evidence types
- Evidence coverage matches the designed distribution exactly:
  - AC (Access Control): all 3 evidence types covered — no gaps
  - CM (Change Management): all 3 evidence types covered — no gaps
  - DQ (Data Quality): 1 of 2 evidence types covered — gap is `data_transform_test`
  - IH (Incident Handling): 1 of 3 evidence types covered — gaps are `incident_response_plan` and `postmortem_record`
- All file paths are relative — no absolute paths
- Zero PII, secrets, or real Synchrony identifiers
- File is UTF-8 encoded

- [ ] T004 [P] [FR-022] Create `sample_test_catalog.json` at `.cursor/skills/rcsa/assets/sample-data/sample_test_catalog.json`

**Acceptance Criteria for T004**:
- File parses with `json.load()` without error
- Contains a `_meta` object with `"scale": "demo"`
- Contains 5–10 test entries
- Each test has at minimum: a test identifier, a description, a file path (relative), and a `controls_relevant` field mapping to control IDs from the control library
- Test entries reinforce the same coverage distribution as the artifact index (tests exist for covered evidence types; no tests exist for gap areas)
- All file paths are relative — no absolute paths
- Zero PII, secrets, or real Synchrony identifiers
- File is UTF-8 encoded

**Checkpoint**: All 3 JSON files (control library, artifact index, test catalog) parse without error. Artifact index has 10–20 entries, test catalog has 5–10 entries. Coverage gaps for DQ and IH are present by design. *(Satisfies AS-01, AS-03, AS-04)*

---

## Phase 4: Citation Format

**Purpose**: Define the citation syntax that templates (Phase 5) use for placeholders and that F6 scripts will generate at runtime. This must be complete before templates are authored so they use the correct citation syntax.

- [ ] T005 [FR-021] Create `citation_format.md` at `.cursor/skills/rcsa/assets/templates/citation_format.md`

**Acceptance Criteria for T005**:
- Defines the citation syntax: `[file_path — description]` with em dash ` — ` as separator
- Includes resolution rules explaining how a citation maps back to an artifact in the artifact index
- Provides worked examples for at least 2 different controls (e.g., one AC citation and one DQ citation)
- Covers edge cases: what a citation looks like when evidence is missing (gap scenario)
- Written in plain language understandable to a non-developer stakeholder
- File is UTF-8 encoded
- Zero PII, secrets, or real Synchrony identifiers

**Checkpoint**: Citation format spec is complete with syntax definition, resolution rules, and ≥2 worked examples. *(Satisfies AS-08)*

---

## Phase 5: Templates

**Purpose**: Create the two Markdown templates that F5 (SKILL.md) will instruct the LLM to populate and F6 (Scripts) will validate. Both templates depend on the control library (T002) for control names/IDs and the citation format (T005) for placeholder syntax. They are independent of each other and can be built in parallel.

- [ ] T006 [P] [FR-021] Create `rcsa_control_narratives_template.md` at `.cursor/skills/rcsa/assets/templates/rcsa_control_narratives_template.md`

**Acceptance Criteria for T006**:
- Contains valid YAML front matter (parses without error)
- Includes a summary table listing all 4 controls with columns for status and key findings
- Contains 4 per-control sections — one each for AC, CM, DQ, and IH
- Each per-control section includes:
  - A narrative area with citation placeholders using the `[file_path — description]` syntax from `citation_format.md`
  - A gap callout area clearly marked for flagging missing evidence
- All placeholders use a single consistent syntax (e.g., `{{field_name}}`) that F6 can programmatically find-and-replace
- Written in plain language understandable to a non-developer stakeholder
- File is UTF-8 encoded
- Zero PII, secrets, or real Synchrony identifiers

- [ ] T007 [P] [FR-021] Create `validation_report_template.md` at `.cursor/skills/rcsa/assets/templates/validation_report_template.md`

**Acceptance Criteria for T007**:
- Contains valid YAML front matter (parses without error)
- Includes a citation index table (columns for citation reference, file path, resolution status)
- Includes an evidence coverage table (columns for control ID, evidence type, covered/gap status)
- Includes a flagged-for-review section for items requiring human attention
- All placeholders use the same consistent syntax as the narratives template (e.g., `{{field_name}}`) — F6 must be able to use one find-and-replace pattern across both templates
- Written in plain language understandable to a non-developer stakeholder
- File is UTF-8 encoded
- Zero PII, secrets, or real Synchrony identifiers

**Checkpoint**: Both templates have valid YAML front matter, correct section structure, citation placeholders matching the citation format spec, and consistent `{{field_name}}` placeholder syntax across both files. *(Satisfies AS-06, AS-07, AS-12)*

---

## Phase 6: Verification & Final Review

**Purpose**: Verify that all 6 files work together as a coherent asset package. This phase covers evidence coverage verification (confirming the designed gap pattern) and the security/encoding sweep across all files.

**⚠️ CRITICAL**: Both tasks in this phase depend on all prior phases being complete.

- [ ] T008 [FR-022] Verify evidence coverage across control library, artifact index, and test catalog

**Acceptance Criteria for T008**:
- Cross-reference `sample_artifact_index.json` and `sample_test_catalog.json` against `sample_control_library.json`
- Confirm coverage matches the designed distribution exactly:
  - AC (Access Control): 3 of 3 evidence types covered — full coverage
  - CM (Change Management): 3 of 3 evidence types covered — full coverage
  - DQ (Data Quality): 1 of 2 evidence types covered — `data_transform_test` is a gap
  - IH (Incident Handling): 1 of 3 evidence types covered — `incident_response_plan` and `postmortem_record` are gaps
- No evidence type is accidentally covered that should be a gap
- No evidence type is accidentally missing that should be covered
- This verification may use a Python 3.11 script as a development-time check (tier 1 validation only — the script is not a deliverable and must not be committed as a runtime dependency)

- [ ] T009 [P] Security and encoding review of all 6 files in `.cursor/skills/rcsa/assets/`

**Acceptance Criteria for T009**:
- All 6 files are valid UTF-8
- Zero PII across all files
- Zero API keys, credentials, or secrets across all files
- Zero absolute file paths across all files
- Zero real Synchrony identifiers across all files
- Asset directory contains exactly 6 files — no extra, no missing:
  1. `sample-data/sample_control_library.json`
  2. `sample-data/sample_artifact_index.json`
  3. `sample-data/sample_test_catalog.json`
  4. `templates/citation_format.md`
  5. `templates/rcsa_control_narratives_template.md`
  6. `templates/validation_report_template.md`

**Checkpoint**: Evidence coverage matches design exactly with intended gaps in DQ and IH. All 6 files pass security and encoding checks. Asset directory contains exactly the expected files with no extras. *(Satisfies AS-05, AS-09, AS-10, AS-11)*

---

## Dependencies & Execution Order

### Phase Dependencies

- **Phase 1 (Setup)**: No dependencies — can start immediately
- **Phase 2 (Foundational)**: Depends on Phase 1 — BLOCKS all subsequent phases
- **Phase 3 (Sample Data)**: Depends on Phase 2 — T003 and T004 can run in parallel
- **Phase 4 (Citation Format)**: Depends on Phase 2 — can run in parallel with Phase 3
- **Phase 5 (Templates)**: Depends on Phase 2 and Phase 4 — T006 and T007 can run in parallel
- **Phase 6 (Verification)**: Depends on all prior phases being complete

### Task Dependencies

| Task | Depends On | Can Parallel With |
|------|-----------|-------------------|
| T001 | — | — |
| T002 | T001 | — |
| T003 | T002 | T004, T005 |
| T004 | T002 | T003, T005 |
| T005 | T002 | T003, T004 |
| T006 | T002, T005 | T007 |
| T007 | T002, T005 | T006 |
| T008 | T002, T003, T004 | T009 |
| T009 | All prior tasks | T008 |

### Post-F4 Gates

These gates are **not F4 implementation tasks**. They are event-driven checkpoints that occur after F4 is complete, when downstream features reach specific milestones.

| Gate | Trigger | Action |
|------|---------|--------|
| G2.1 — F5 Alignment Check | F5 build complete | Compare F5's LLM output instructions against F4's template fields and citation format. Reconcile any mismatches. |
| G2.2 — F6 Compatibility Check | F6 plan approved | Compare F6's input expectations against F4's JSON schemas and sample data. **Blocking gate** — F6 implementation cannot start until this passes. |

---

## Implementation Strategy

### Sequential (Single Implementer)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational (control library) — **blocks everything**
3. Complete Phase 3: Sample Data (artifact index, then test catalog)
4. Complete Phase 4: Citation Format
5. Complete Phase 5: Templates (narratives template, then validation report template)
6. Complete Phase 6: Verification & Final Review
7. **STOP and VALIDATE**: All 12 acceptance scenarios satisfied
8. Commit all 6 files to repository

### With Parallel Opportunities

1. Complete Phase 1 + Phase 2 sequentially
2. Once Phase 2 is done, launch Phase 3 (T003, T004) and Phase 4 (T005) in parallel
3. Once Phase 4 is done, launch Phase 5 (T006, T007) in parallel
4. Once all files exist, run Phase 6 (T008, T009) in parallel
5. **STOP and VALIDATE**: All 12 acceptance scenarios satisfied
6. Commit all 6 files to repository

---

## Traceability Matrix

### Acceptance Scenario → Task Mapping

| Acceptance Scenario | Description | Task(s) |
|---------------------|-------------|---------|
| AS-01 | All JSON files parse without error | T002, T003, T004 |
| AS-02 | Control library contains all 4 controls | T002 |
| AS-03 | Artifact index provides demo-scale evidence | T003 |
| AS-04 | Test catalog provides demo-scale tests | T004 |
| AS-05 | Evidence coverage matches design exactly | T008 |
| AS-06 | Narratives template is structurally valid | T006 |
| AS-07 | Validation report template is structurally valid | T007 |
| AS-08 | Citation format spec is complete | T005 |
| AS-09 | No sensitive data in any file | T009 |
| AS-10 | All files are UTF-8 encoded | T009 |
| AS-11 | Asset directory contains exactly the expected files | T001, T009 |
| AS-12 | Template placeholders use consistent syntax | T006, T007 |

### Task → Acceptance Scenario Mapping

| Task | Acceptance Scenarios |
|------|---------------------|
| T001 | AS-11 |
| T002 | AS-01, AS-02 |
| T003 | AS-01, AS-03 |
| T004 | AS-01, AS-04 |
| T005 | AS-08 |
| T006 | AS-06, AS-12 |
| T007 | AS-07, AS-12 |
| T008 | AS-05 |
| T009 | AS-09, AS-10, AS-11 |

---

## Notes

- **F4 is static assets only** — no runtime code. Any Python used for verification (T008) is a development-time tool, not a deliverable.
- **All sample data is fictional** — designed for demo purposes. Zero PII, real identifiers, or sensitive information.
- **Placeholder syntax must be consistent** — both templates (T006, T007) must use the same `{{field_name}}` pattern so F6 can use a single find-and-replace approach.
- **Coverage gaps are intentional** — DQ missing `data_transform_test` and IH missing `incident_response_plan` + `postmortem_record` are by design to demonstrate gap detection in F5/F6.
- **Commit after Phase 6 passes** — do not commit individual files incrementally. All 6 files should be committed together once all acceptance scenarios are satisfied.
- **Post-F4 gates (G2.1, G2.2) are not part of this task list** — they are triggered by downstream feature milestones and documented here for awareness only.
