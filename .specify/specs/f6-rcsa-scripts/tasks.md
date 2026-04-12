# Tasks: F6 — RCSA Scripts (Deterministic Python Pipeline)

**Input**: Design documents from `/specs/f6-rcsa-scripts/`
**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, F6_F4_contracts.md, F6_F5_contracts.md, script_conventions.md

**Tests**: Tests are included. Tier 1 unit tests are inline with each script task (TDD — write tests first, ensure they fail before implementation). Tier 2 and Tier 3 tests are in the Polish phase.

**Organization**: F6 serves a single user story (US2 — Generate RCSA Control Narratives, P1). All tasks are grouped under US2. Scripts are ordered by pipeline execution sequence.

## Format: `[ID] [Story] Description`

- **[Story]**: Which user story this task belongs to (US2 for all F6 tasks)
- Include exact file paths in descriptions

## Path Conventions

- **Scripts**: `.cursor/skills/rcsa/scripts/`
- **Tests**: `tests/rcsa/scripts/`
- **Output**: `.cursor/skills/rcsa/output/` (created at runtime by `write_output.py`)
- **Input data**: `.cursor/skills/rcsa/data/` (F4 — read only)
- **Spec docs**: `.specify/specs/f6-rcsa-scripts/`

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and directory structure per plan.md

- [ ] T001 [US2] Create script directory structure: `.cursor/skills/rcsa/scripts/` for the 6 Python scripts
- [ ] T002 [US2] Create test directory structure: `tests/rcsa/scripts/unit/` for Tier 1 unit tests and `tests/rcsa/scripts/integration/` for Tier 2 integration tests
- [ ] T003 [US2] Create `.gitkeep` in `.cursor/skills/rcsa/output/` to track the output directory in version control (contents are gitignored)
- [ ] T004 [US2] Add `requirements.txt` at `.cursor/skills/rcsa/scripts/requirements.txt` — Python 3.11+, no external dependencies beyond stdlib for demo (json, re, sys, os, pathlib, datetime)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core conventions and references that MUST be complete before ANY script implementation can begin

**⚠️ CRITICAL**: No script implementation can begin until this phase is complete

- [ ] T005 [US2] Add `script_conventions.md` to `.specify/specs/f6-rcsa-scripts/` — defines exit codes, error message format, logging conventions, citation regex, confidence tiers, data flow, and common Python patterns. All script tasks reference this document. *(File created — commit to repo)*
- [ ] T006 [US2] Verify F4 sample data files exist at `.cursor/skills/rcsa/data/`: `sample_control_library.json`, `sample_artifact_index.json`, `sample_test_catalog.json`. Confirm they match the schemas defined in `F6_F4_contracts.md` and `data-model.md` Section 1. If files are missing or mismatched, resolve with F4 before proceeding.
- [ ] T007 [US2] Verify F4 citation format alignment: confirm `citation_format.md` in F4 specifies `[file_path — description]` with em dash delimiter (U+2014). This is the format `validate_citations.py` will parse. If misaligned, resolve with F4 before proceeding.

**Checkpoint**: Foundation ready — script implementation can now begin in pipeline order

---

## Phase 3: User Story 2 — Generate RCSA Control Narratives (Priority: P1) 🎯 MVP

**Goal**: Implement the 6-script deterministic Python pipeline that validates input, builds an artifact registry, maps evidence to controls, formats the LLM prompt, validates LLM citations, and writes the final output files.

**Independent Test**: Run the full pipeline with F4's sample data and a mocked LLM response. The pipeline should produce `rcsa_control_narratives.md` and `validation_report.md` in `.cursor/skills/rcsa/output/` with correct content for all 4 demo controls (AC, CM, DQ, IH).

### Tests for User Story 2 ⚠️

> **NOTE: Write these tests FIRST for each script, ensure they FAIL before implementation**

Tier 1 unit tests are listed immediately before each script's implementation task. Each test file covers the acceptance scenarios defined in spec.md and the validation rules defined in data-model.md.

### Script 1: validate_input.py

- [ ] T008 [US2] Write Tier 1 unit tests in `tests/rcsa/scripts/unit/test_validate_input.py`. Test cases:
  - Valid input: all 3 files present, valid JSON, all required fields, all constraints pass → `status: "pass"`
  - Missing file: one or more JSON files not found → `status: "fail"` with `FILE_NOT_FOUND` error
  - Invalid JSON: file exists but is not valid JSON → `status: "fail"` with `PARSE_ERROR`
  - Missing required field: `controls[].objective` absent → `status: "fail"` with `SCHEMA_ERROR`
  - Invalid control ID: `controls[].id` does not match `^[A-Z]{2,4}$` → `status: "fail"` with `VALIDATION_ERROR`
  - Duplicate IDs: two controls with same `id` → `status: "fail"` with `VALIDATION_ERROR`
  - Invalid test reference: `tests[].controls_relevant` contains ID not in control library → `status: "fail"` with `VALIDATION_ERROR`
  - Invalid artifact ID format: `artifacts[].id` does not match `ART-NNN` → `status: "fail"` with `VALIDATION_ERROR`
  - Invalid test ID format: `tests[].id` does not match `TST-NNN` → `status: "fail"` with `VALIDATION_ERROR`
  - Empty arrays: `controls`, `artifacts`, or `tests` is empty → `status: "fail"` with `SCHEMA_ERROR`
  - Evidence types empty: a control has empty `evidence_types` array → `status: "fail"` with `SCHEMA_ERROR`
  - All tests must verify error messages follow `script_conventions.md` Section 2 format

- [ ] T009 [US2] Implement `validate_input.py` in `.cursor/skills/rcsa/scripts/validate_input.py`. Receives file paths to 3 JSON files as arguments. Returns `ValidationResult` (schema: `data-model.md` Section 2.1). Applies all validation rules from `data-model.md` Section 2.1 validation rules table. Follows exit code, error format, and logging conventions from `script_conventions.md`. Logs to stderr, returns JSON to agent runtime.

### Script 2: build_registry.py

- [ ] T010 [US2] Write Tier 1 unit tests in `tests/rcsa/scripts/unit/test_build_registry.py`. Test cases:
  - Valid input: 3 JSON files with demo data → returns `ArtifactRegistry` with 15 artifacts, 8 tests, 23 file_path_index entries
  - Artifact lookup: `registry.artifacts["ART-001"]` returns correct artifact object with `source: "artifact_index"`
  - Test lookup: `registry.tests["TST-001"]` returns correct test object with `source: "test_catalog"`
  - File path index: `registry.file_path_index["auth/oauth_config.yaml"]` returns `"ART-001"`
  - File path index includes tests: `registry.file_path_index["tests/auth/test_rbac_permissions.py"]` returns `"TST-001"`
  - Stats accuracy: `stats.total_artifacts` = 15, `stats.total_tests` = 8, `stats.total_entries` = 23
  - All entries have `source` field set correctly

- [ ] T011 [US2] Implement `build_registry.py` in `.cursor/skills/rcsa/scripts/build_registry.py`. Receives file paths to 3 JSON files as arguments. Returns `ArtifactRegistry` (schema: `data-model.md` Section 2.2). Builds O(1) lookup dictionaries keyed by ID. Builds `file_path_index` reverse lookup (file_path → ID) for citation validation. Follows conventions from `script_conventions.md`.

### Script 3: map_evidence.py

- [ ] T012 [US2] Write Tier 1 unit tests in `tests/rcsa/scripts/unit/test_map_evidence.py`. Test cases:
  - AC mapping: 5 artifacts mapped, 3 tests mapped, all 3 evidence types covered, `gap_flag: false`
  - CM mapping: 6 artifacts mapped, 3 tests mapped, all 3 evidence types covered, `gap_flag: false`
  - DQ mapping: 2 artifacts mapped, 1 test mapped, `evidence_type_coverage.data_transform_test` is empty, `gap_flag: false` (has artifacts, just missing one evidence type)
  - IH mapping: 2 artifacts mapped, 1 test mapped, `evidence_type_coverage.incident_response_plan` and `postmortem_record` are empty, `gap_flag: false`
  - Gap detection: control with zero artifacts AND zero tests → `gap_flag: true`
  - Summary stats: `total_controls: 4`, `controls_with_evidence: 4`, `controls_with_gaps: 0`, `total_artifacts_mapped: 15`, `total_tests_mapped: 8`
  - Unmapped tracking: `unmapped_artifacts` and `unmapped_tests` arrays are empty for demo data
  - Evidence type matching: each artifact's `evidence_type_matched` field is correct

- [ ] T013 [US2] Implement `map_evidence.py` in `.cursor/skills/rcsa/scripts/map_evidence.py`. Receives `ArtifactRegistry` and control library data as arguments. Returns `MappedEvidence` (schema: `data-model.md` Section 2.3). Maps artifacts to controls by matching artifact descriptions/file paths against evidence type descriptions. Maps tests to controls via `controls_relevant[]`. Sets `gap_flag: true` when a control has zero artifacts AND zero tests. Builds `evidence_type_coverage` showing which evidence types have artifacts and which are empty. Follows conventions from `script_conventions.md`.

  **Implementation note** (from `data-model.md`): The artifact index does NOT contain an `evidence_type` field. The mapping must be inferred from the artifact's `description` and `file_path` against the control library's `evidence_types[].description`. For the demo, implement keyword/path-based heuristics. This is the one step where the mapping logic must be carefully designed — see `data-model.md` Section 2.3 "Note on artifact-to-evidence-type matching."

### Script 4: orchestrate_llm.py

- [ ] T014 [US2] Write Tier 1 unit tests in `tests/rcsa/scripts/unit/test_orchestrate_llm.py`. Test cases:
  - **Format mode** (`--mode format`):
    - Valid input: `MappedEvidence` with 4 controls → returns formatted prompt string containing all control IDs, objectives, artifact descriptions, and test descriptions
    - Prompt includes citation format instructions: tells LLM to use `[file_path — description]` format
    - Prompt includes confidence tier rubric: HIGH, MEDIUM, LOW, GAP
    - Prompt includes all mapped artifacts and tests per control
    - Gap controls: control with `gap_flag: true` → prompt instructs LLM to produce `[GAP]` narrative
  - **Parse mode** (`--mode parse`):
    - Valid LLM response: Markdown with 4 control sections → returns `LLMResponse` with `status: "success"` and 4 narratives
    - Confidence extraction: `**Confidence**: HIGH` line → `confidence: "HIGH"` in parsed output
    - Citation extraction: `[auth/oauth_config.yaml — OAuth2 provider configuration]` → `artifacts_cited` includes `"ART-001"`
    - Missing control section: LLM omits one control → that control gets `confidence: "UNAVAILABLE"`, others parse normally
    - Missing confidence line: section exists but no `**Confidence**:` line → defaults to `"UNAVAILABLE"`
    - Empty LLM response: empty string input → returns `LLMResponse` with `status: "degraded"`
    - Malformed LLM response: unparseable content → returns `LLMResponse` with `status: "degraded"`, `mapped_evidence_passthrough` populated
    - YAML front matter: parsed response includes correct `yaml_front_matter` with `controls_assessed`, `controls_passing`, `controls_gap` counts

- [ ] T015 [US2] Implement `orchestrate_llm.py` in `.cursor/skills/rcsa/scripts/orchestrate_llm.py`. Dual-mode script controlled by `--mode` flag:
  - **`--mode format`**: Receives `MappedEvidence`. Returns a formatted prompt string for the LLM. Prompt includes per-control evidence summaries, citation format instructions (`[file_path — description]`), confidence tier rubric, and output structure instructions (Markdown with `## [CONTROL_ID] — [Control Name]` headings, `**Confidence**: {TIER}` lines, `---` separators).
  - **`--mode parse`**: Receives raw LLM response (Markdown string) and `MappedEvidence`. Parses control sections by splitting on `## [CONTROL_ID] —` headings. Extracts confidence tier from `**Confidence**: {TIER}` line. Extracts citations using regex from `script_conventions.md` Section 6. Returns `LLMResponse` (schema: `data-model.md` Section 2.4). Handles degraded mode per `F6_F5_contracts.md` Section 5.
  - F6 never calls the LLM directly. F5 owns LLM invocation. Follows conventions from `script_conventions.md`.

### Script 5: validate_citations.py

- [ ] T016 [US2] Write Tier 1 unit tests in `tests/rcsa/scripts/unit/test_validate_citations.py`. Test cases:
  - All citations resolve: 8 citations, all match `file_path_index` entries → `resolution_rate: 1.0`
  - Hallucinated citation: `[nonexistent/file.py — some description]` → `resolved: false`, appears in `unresolved_list`
  - Mixed results: some resolve, some don't → correct counts for `resolved`, `unresolved`, `resolution_rate`
  - No citations: LLM produced no citations → `total_citations: 0`, `resolution_rate: 0.0`
  - Citation regex: correctly parses `[file_path — description]` with em dash (U+2014), not hyphen
  - Per-citation detail: each citation entry has correct `citation_text`, `file_path_extracted`, `resolved_to`, `control_id`
  - Degraded LLM response: `LLMResponse` with `status: "degraded"` → returns empty `CitationValidationResult` with zero citations

- [ ] T017 [US2] Implement `validate_citations.py` in `.cursor/skills/rcsa/scripts/validate_citations.py`. Receives `LLMResponse` and `ArtifactRegistry` as arguments. Returns `CitationValidationResult` (schema: `data-model.md` Section 2.5). Extracts citations using regex `\[([^\]]+?)\s—\s([^\]]+?)\]`. Resolves `file_path_extracted` against `registry.file_path_index`. Builds per-citation detail and `unresolved_list`. Unresolved citations are reported — they do NOT stop the pipeline. Follows conventions from `script_conventions.md`.

### Script 6: write_output.py

- [ ] T018 [US2] Write Tier 1 unit tests in `tests/rcsa/scripts/unit/test_write_output.py`. Test cases:
  - **`rcsa_control_narratives.md`**:
    - YAML front matter: contains `title`, `generated` (ISO 8601), `controls_assessed: 4`, correct `controls_passing`, `controls_gap` counts
    - Summary table: one row per control with correct Control ID, Control Name, Confidence, Artifacts count, Tests count, Status
    - Per-control sections: heading `## AC — Access Control`, confidence line, narrative text present
    - All 4 controls present in output
  - **`validation_report.md`**:
    - Citation resolution table: one row per citation with correct Artifact ID, File Path, Resolved status, Control
    - Totals: correct Total Citations, Resolved, Unresolved, Resolution Rate
    - Evidence coverage table: one row per control with correct Artifacts, Tests, Evidence Types Covered, Evidence Types Missing
    - Warnings section: lists DQ missing `data_transform_test`, IH missing `incident_response_plan` and `postmortem_record`
  - **Degraded mode**:
    - YAML front matter includes `mode: "degraded"`
    - Summary table shows `UNAVAILABLE` for degraded controls
    - Raw evidence listing instead of narratives for degraded controls
  - **File operations**:
    - Creates `output/` directory if it doesn't exist
    - Writes UTF-8 encoded files with LF line endings
    - Overwrites existing files without prompting

- [ ] T019 [US2] Implement `write_output.py` in `.cursor/skills/rcsa/scripts/write_output.py`. Receives `LLMResponse`, `CitationValidationResult`, and `MappedEvidence` as arguments. Writes two files to `.cursor/skills/rcsa/output/`:
  - `rcsa_control_narratives.md`: YAML front matter + summary table + per-control narrative sections (schema: `data-model.md` Section 3.1)
  - `validation_report.md`: citation resolution table + evidence coverage table + warnings (schema: `data-model.md` Section 3.2)
  - Creates output directory at runtime if it doesn't exist. Handles degraded mode per `F6_F5_contracts.md` Section 5. Returns output file paths to agent runtime. Follows conventions from `script_conventions.md`.

**Checkpoint**: At this point, all 6 scripts should be individually functional with passing Tier 1 unit tests. Each script can be tested independently with sample data.

---

## Phase 4: Polish & Cross-Cutting Concerns

**Purpose**: Integration testing, documentation, and validation across all scripts

- [ ] T020 [US2] Write Tier 2 integration tests in `tests/rcsa/scripts/integration/test_pipeline_integration.py`. Test the full 6-script pipeline end-to-end using F4's sample data and a **mocked** LLM response (hardcoded Markdown string matching the expected format from `F6_F5_contracts.md` Section 3). Verify:
  - `validate_input.py` passes → `build_registry.py` produces correct registry → `map_evidence.py` maps all 15 artifacts and 8 tests → `orchestrate_llm.py` formats prompt containing all 4 controls → `orchestrate_llm.py` parses mocked response → `validate_citations.py` resolves all citations → `write_output.py` produces both output files
  - Output files match expected content for all 4 demo controls
  - Degraded mode integration: pass empty LLM response through the pipeline → verify degraded output files are produced correctly

- [ ] T021 [US2] Write Tier 3 end-to-end test plan in `tests/rcsa/scripts/integration/test_e2e_plan.md`. Document the test procedure for running the full pipeline with F5's SKILL.md and a live LLM. This test cannot be executed until F5 is complete. Include: prerequisites, setup steps, expected inputs, expected outputs, pass/fail criteria. **⚠️ Blocked by F5**

- [ ] T022 [US2] Run `quickstart.md` validation: follow the quickstart guide step-by-step and verify every command works, every expected output is produced, and every file path is correct. Update `quickstart.md` if any steps are inaccurate.

- [ ] T023 [US2] Code review and cleanup across all 6 scripts: verify consistent error message format, logging format, exit code usage, and JSON output schemas match `data-model.md` and `script_conventions.md`. Remove any debug code or TODO comments.

- [ ] T024 [US2] Update `plan.md` status from Draft to Complete. Add any implementation notes or deviations discovered during development.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion — BLOCKS all script implementation
- **User Story 2 (Phase 3)**: Depends on Foundational phase completion. Scripts are implemented in strict pipeline order.
- **Polish (Phase 4)**: Depends on all Phase 3 scripts being complete

### Within Phase 3 (Script Implementation Order)

Scripts are implemented in strict pipeline order. Each script depends on the previous script being complete because:
- Later scripts consume the output schemas of earlier scripts
- Understanding the upstream data shapes is necessary to implement downstream logic
- Tier 1 tests for each script use the output of the previous script as test fixtures

| Order | Script | Depends On |
|---|---|---|
| 1 | `validate_input.py` | Phase 2 complete |
| 2 | `build_registry.py` | `validate_input.py` complete |
| 3 | `map_evidence.py` | `build_registry.py` complete |
| 4 | `orchestrate_llm.py` | `map_evidence.py` complete |
| 5 | `validate_citations.py` | `orchestrate_llm.py` complete |
| 6 | `write_output.py` | `validate_citations.py` complete |

### Within Each Script

- Tier 1 unit tests MUST be written and FAIL before implementation
- Implementation makes the tests pass
- Script complete = all Tier 1 tests passing

### Cross-Feature Dependencies

| Dependency | Status | Impact |
|---|---|---|
| F4 sample data files | Locked | Required for Phase 2 verification (T006) and all testing |
| F4 citation format | Locked | Required for Phase 2 verification (T007) and `validate_citations.py` |
| F5 SKILL.md | Not yet complete | Blocks Tier 3 end-to-end testing (T021) only — all other work is independent |

---

## Implementation Strategy

### MVP First

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational (CRITICAL — blocks all scripts)
3. Complete Phase 3: All 6 scripts in pipeline order, TDD style
4. **STOP and VALIDATE**: Run Tier 2 integration tests with mocked LLM
5. Complete Phase 4: Polish, Tier 3 test plan, quickstart validation

### Commit Strategy

- Commit after each script + its passing Tier 1 tests (one commit per script pair: T008+T009, T010+T011, etc.)
- Commit Tier 2 integration tests separately
- Commit polish tasks individually

---

## Notes

- All tasks are `[US2]` — F6 serves only User Story 2 (Generate RCSA Control Narratives)
- No `[P]` parallel markers — scripts are implemented in strict pipeline order
- Tier 1 tests are TDD: write test → verify it fails → implement script → verify it passes
- `orchestrate_llm.py` is one script with two modes (`--mode format` and `--mode parse`)
- F6 never calls the LLM directly — F5 owns LLM invocation
- All schemas referenced in task descriptions are defined in `data-model.md`
- All conventions referenced in task descriptions are defined in `script_conventions.md`
- All contract obligations are defined in `F6_F4_contracts.md` and `F6_F5_contracts.md`
