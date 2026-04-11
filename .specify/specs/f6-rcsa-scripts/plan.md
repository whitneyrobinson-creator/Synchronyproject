# Implementation Plan: F6 — RCSA Scripts (Deterministic Python Pipeline)

**Branch**: `f6-rcsa-scripts` | **Date**: 2026-04-11 | **Spec**: `/specs/f6-rcsa-scripts/spec.md`

**Input**: Feature specification from `/specs/f6-rcsa-scripts/spec.md`

**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/plan-template.md` for the execution workflow.

---

## Summary

F6 delivers the deterministic Python pipeline that wraps the RCSA Control Narrative Generation skill, executing both before and after the LLM reasoning step (F5). The pipeline consists of six Python 3.11 modules under `.cursor/skills/rcsa/scripts/`, invoked individually by the SKILL.md agent — there is no standalone pipeline runner; F5 orchestrates the script sequence:

1. **Pre-LLM**: `validate_input.py` → `build_registry.py` → `map_evidence.py`
2. **LLM reasoning** (F5)
3. **Post-LLM**: `validate_citations.py` → `write_output.py`
4. **LLM orchestration**: `orchestrate_llm.py` (handles invocation formatting, response parsing, and graceful degradation)

Pre-LLM, the scripts validate all three JSON input files (artifact index, test catalog, control library) against F4's schemas, build an in-memory artifact registry (no intermediate persistence — acceptable for demo scale), and produce a structured mapped evidence JSON that the LLM consumes. Post-LLM, the scripts validate every inline citation against the artifact registry to detect hallucinations, then populate F4's templates and write two Markdown deliverables: `rcsa_control_narratives.md` and `validation_report.md`.

The pipeline implements 12 functional requirements (FR-001 through FR-005, FR-007 through FR-013) across 5 user stories (4× P1, 1× P2), all targeting the May 7th demo. It enforces Constitution Principle 1 (Accuracy Over Speed) through citation validation and Principle 2 (Graceful Degradation) by producing raw evidence output when the LLM is unavailable — the scripts never crash due to LLM failure. Storage is local file-based only. The pipeline targets < 60 seconds end-to-end for demo-scale inputs (10–20 artifacts, 5–10 tests, 4 controls).

F6 is the final feature in the build order (after F5 and F4) and depends on F4's locked JSON schemas, template placeholder syntax, and citation format spec. Pre-LLM and post-LLM steps are independently testable using F4's sample data and mocked LLM responses. Full end-to-end testing with a live LLM requires F5 integration.

---

## Technical Context

**Language/Version**: Python 3.11 (Constitution Tech Stack requirement)

**Primary Dependencies**: Python standard library only — zero third-party dependencies.

- `json` — JSON parsing and validation
- `os` / `pathlib` — File system operations, directory creation
- `datetime` — Timestamps for logging and output metadata
- `logging` — Structured logging for audit trail (FR-013)
- `re` — Citation pattern matching during validation

**Storage**: File-based only. JSON intermediates and Markdown outputs written to local filesystem. No database, no cloud storage (Constitution constraint).

**LLM Invocation**: F6 scripts do not call any API. The Cursor agent runtime handles all LLM communication natively. `orchestrate_llm.py` formats the mapped evidence for the SKILL.md to consume and parses the LLM's response. Retry logic and API-level error handling are the agent runtime's responsibility. F6 handles graceful degradation when the LLM response is missing, empty, or malformed.

**Testing**: Three-tier validation:

- **Tier 1 (unit)**: Each script tested independently — valid/invalid inputs, expected outputs, error handling. No LLM required.
- **Tier 2 (integration with mocks)**: Full pipeline run with mocked LLM responses (success, malformed, empty). Verifies graceful degradation paths.
- **Tier 3 (end-to-end)**: Full pipeline with live LLM via F5 in Cursor. Requires F5 integration.

**Target Platform**: Local development environment (Cursor IDE, PowerShell)

**Project Type**: Deterministic Python pipeline (6 scripts, no services, no API calls)

**Performance Goals**:

| Metric | Target | Notes |
|---|---|---|
| Input validation (3 files) | < 5 seconds | Deterministic |
| Registry build + evidence mapping | < 10 seconds | Deterministic |
| LLM data formatting | < 5 seconds | Excludes LLM thinking time |
| Citation validation | < 5 seconds | Deterministic |
| Output file writing | < 5 seconds | Deterministic |
| **Full pipeline (deterministic portions)** | **< 60 seconds** | LLM thinking time is additive, governed by F5's budget |

**Logging**: All processing steps and errors logged via Python `logging` module (FR-013). Logs written to stdout for demo.

**Constraints**:

- Python stdlib only — zero third-party dependencies
- All file I/O uses UTF-8 encoding
- Only four controls supported: AC, CM, DQ, IH — others rejected with error
- Demo-scale only: 10–20 artifacts, 5–10 tests, 4 controls
- No intermediate persistence of artifact registry (in-memory only)

**Scale/Scope**: 6 Python scripts, 12 functional requirements, demo-scale inputs

---

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Evidence |
|---|---|---|
| **Principle 1: Accuracy Over Speed** | ✅ Pass | F6 validates every LLM citation against the artifact registry before output — hallucinated citations are flagged, not silently included. Gap flags have zero false negatives. Ambiguous evidence excluded and flagged for human review. |
| **Principle 2: Graceful Degradation** | ✅ Pass | If LLM response is missing, empty, or malformed, F6 produces raw evidence output using templates populated with extracted data only — no crash, no silent failure. Pre-LLM scripts function independently of LLM availability. |
| **Principle 3: Simplicity First** | ✅ Pass | Zero third-party dependencies. Six single-responsibility scripts. No API calls — Cursor agent runtime handles LLM communication. No database, no services, no build step beyond Python stdlib. |
| **Principle 4: Audit-Ready by Default** | ✅ Pass | Every processing step logged (FR-013). Citation validation produces `validation_report.md` with resolution statistics. Output documents include YAML front matter with timestamps, confidence tiers, and gap flags. Full traceability from input artifacts to output citations. |

| Gate | Status | Evidence |
|---|---|---|
| **Max 3 projects per repo** | ✅ Pass | F6 is one feature within the existing repo structure. |
| **Never-Ever Rules** | ✅ Pass | No API keys in code (no API calls at all). No PII in sample data. All file paths relative. No real Synchrony identifiers. No secrets to commit. |
| **Security: Data Sent to LLM** | ✅ Pass | F6 scripts process data locally. `orchestrate_llm.py` formats mapped evidence as file paths + content snippets only — never raw file contents. This constraint is enforced at the F6 level. Content structure sent to LLM is further controlled by F5's instructions. |
| **Out of Scope Boundaries** | ✅ Pass | No microservices, cloud deployment, frontend UI, standalone API, real-time processing, or agent harness libraries. No repository scanning, multi-use-case documentation, or deployment features. No direct API calls. |
| **Tech Stack Compliance** | ✅ Pass | Python 3.11 stdlib only. Markdown and JSON for data. No additional languages, frameworks, or third-party packages. |
| **Handoff Boundary** | ✅ Pass | F6 scripts are part of the deliverable handed to Synchrony. Scripts are documented (quickstart.md) for Synchrony to run with their own data. Sample data clearly labeled as demo-scale. |
| **Governance: Definition of Done** | ✅ Applicable | F6 is done when: (1) all team members have reviewed, (2) tier 1 and tier 2 tests pass, (3) all acceptance scenarios pass, (4) all team members sign off. Tier 3 (end-to-end with live LLM) is a post-integration validation. |

**Constitution Check Result: PASS — No violations. No complexity justification required.**

---

## Project Structure

### Documentation (this feature)

```text
specs/f6-rcsa-scripts/
├── plan.md              # This file (/speckit.plan command output)
├── research.md          # Phase 0 output (/speckit.plan command)
├── data-model.md        # Phase 1 output (/speckit.plan command)
├── quickstart.md        # Phase 1 output (/speckit.plan command)
├── contracts/           # Phase 1 output (F6↔F4, F6↔F5 interfaces)
└── tasks.md             # Phase 2 output (/speckit.tasks command - NOT created by /speckit.plan)
```

### Source Code (repository root)

```text
.cursor/skills/rcsa/scripts/
├── validate_input.py        # FR-001: Validate JSON inputs against F4 schemas
├── build_registry.py        # FR-002: Build in-memory artifact registry
├── map_evidence.py          # FR-003, FR-004: Map artifacts/tests to controls
├── orchestrate_llm.py       # FR-005, FR-007: Format evidence for LLM, handle degradation
├── validate_citations.py    # FR-008, FR-009: Validate citations, produce report data
└── write_output.py          # FR-010: Populate templates, write Markdown outputs
```

### Test Files (development only — not delivered to Synchrony)

```text
.cursor/skills/rcsa/scripts/tests/
├── test_validate_input.py
├── test_build_registry.py
├── test_map_evidence.py
├── test_orchestrate_llm.py
├── test_validate_citations.py
├── test_write_output.py
└── mocks/
    ├── mock_llm_response_success.md
    ├── mock_llm_response_malformed.md
    └── mock_llm_response_empty.md
```

### Runtime Output (generated per run — not checked into repo)

```text
.cursor/skills/rcsa/output/
├── rcsa_control_narratives.md    # Final audit-ready narrative document
└── validation_report.md          # Citation validation statistics
```

**Structure Decisions**:

1. **Flat scripts directory** — no subdirectories, no `src/` wrapper, no `__init__.py` package structure. Each script is a standalone module invoked individually by the SKILL.md agent. Matches Simplicity First principle and F4's flat asset structure.
2. **Output directory** — `.cursor/skills/rcsa/output/` keeps all skill artifacts under the same root. Created at runtime by `write_output.py`. Must be in `.gitignore`.
3. **No `mapped_evidence.json` on disk** — the mapped evidence data structure is passed in-memory between `map_evidence.py` and `orchestrate_llm.py` via the agent runtime. Consistent with "no intermediate persistence" constraint. If debugging requires inspection, the data is logged to stdout (not written as a file).
4. **Tests directory** — lives alongside scripts for development convenience. Contains mocked LLM responses for tier 2 testing. Not included in Synchrony handoff deliverable.

---

## Complexity Tracking

**No violations found. This section is intentionally empty.**

| Violation | Why Needed | Simpler Alternative Rejected Because |
|---|---|---|
| — | — | — |

---

## Acceptance Scenarios

### AS-001: Happy Path — Full Pipeline Success

**Validates**: FR-001 through FR-010, FR-011, FR-012, FR-013

**Given** valid artifact index, test catalog, and control library JSON files matching F4 schemas

**When** the pipeline runs end-to-end with a well-formed LLM response

**Then**:

- `validate_input.py` exits with success, no errors logged
- `build_registry.py` produces an in-memory registry with all artifacts indexed
- `map_evidence.py` produces mapped evidence covering all 4 controls (AC, CM, DQ, IH)
- `orchestrate_llm.py` formats evidence and receives a valid LLM response
- `validate_citations.py` confirms all citations resolve to registry entries; validation report shows 0 hallucinated citations
- `write_output.py` produces `rcsa_control_narratives.md` and `validation_report.md` in `.cursor/skills/rcsa/output/`
- Both files contain YAML front matter with timestamp, confidence tiers, and gap flags
- Total deterministic processing time < 60 seconds

### AS-002: Invalid Input — Schema Violation

**Validates**: FR-001

**Given** an artifact index JSON file with a missing required field

**When** `validate_input.py` runs

**Then**:

- Script exits with a non-zero exit code
- Error log identifies the specific file, field, and violation
- Pipeline halts — no downstream scripts execute
- No partial output files are created

### AS-003: Invalid Input — Missing File

**Validates**: FR-001

**Given** the test catalog JSON file does not exist at the expected path

**When** `validate_input.py` runs

**Then**:

- Script exits with a non-zero exit code
- Error log identifies the missing file path
- Pipeline halts

### AS-004: Unsupported Control Code

**Validates**: FR-001

**Given** a control library JSON containing a control code "XX" not in {AC, CM, DQ, IH}

**When** `validate_input.py` runs

**Then**:

- Script rejects the unsupported control with a clear error message
- Pipeline halts

### AS-005: Graceful Degradation — LLM Unavailable or Malformed

**Validates**: FR-005, FR-007

**Given** valid inputs and successful pre-LLM processing

**When** the LLM returns no response (empty/null) OR a response that doesn't match the expected structure

**Then**:

- `orchestrate_llm.py` detects the failure and sets a degradation flag
- Pipeline enters raw evidence mode: `write_output.py` produces `rcsa_control_narratives.md` populated with mapped evidence only (no LLM-generated narratives)
- `validation_report.md` notes "LLM unavailable — raw evidence mode" with the failure reason
- Error log captures the raw LLM response (if any) for debugging
- No crash, no unhandled exception

### AS-006: Citation Validation — Hallucinated Citation Detected

**Validates**: FR-008, FR-009

**Given** a valid LLM response containing a citation `[ART-999]` that does not exist in the artifact registry

**When** `validate_citations.py` runs

**Then**:

- The hallucinated citation is flagged with an inline marker: `⚠️ [ART-999] (unresolved)` in the output narrative
- `validation_report.md` shows count of hallucinated vs. valid citations
- Pipeline completes — hallucinated citations are flagged, not silently removed, and don't crash the pipeline

### AS-007: Citation Validation — All Citations Valid

**Validates**: FR-008

**Given** a valid LLM response where every citation resolves to a registry entry

**When** `validate_citations.py` runs

**Then**:

- Validation report shows 100% citation resolution rate
- No flags or warnings in the output

### AS-008: Gap Detection — Control With No Evidence

**Validates**: FR-003, FR-004

**Given** valid inputs where control "DQ" has no matching artifacts or tests

**When** `map_evidence.py` runs

**Then**:

- Mapped evidence for DQ is empty with `gap_flag: true`
- The gap is logged
- `rcsa_control_narratives.md` includes a visible gap indicator: `⚠️ COVERAGE GAP: No evidence found for DQ`
- Zero false negatives — if evidence exists, it must not be missed

### AS-009: Partial Evidence — Control With Incomplete Coverage

**Validates**: FR-003, FR-004

**Given** valid inputs where control "CM" has 3 matching artifacts but 0 matching tests

**When** `map_evidence.py` runs

**Then**:

- Mapped evidence for CM includes the 3 artifacts with `gap_flag: false` but `partial_coverage: true`
- Confidence tier reflects partial coverage (not "high")
- Output narrative notes: "No test evidence found for CM — artifact evidence only"
- Pipeline continues — partial evidence is valid, not an error

### AS-010: Output File Integrity

**Validates**: FR-010, FR-012

**Given** a successful pipeline run

**When** `write_output.py` completes

**Then**:

- Both output files are valid Markdown (parseable, no broken syntax)
- YAML front matter is valid YAML
- All template placeholders from F4 templates are populated — no raw `{{placeholder}}` strings remain
- Files are UTF-8 encoded
- Files are written to `.cursor/skills/rcsa/output/`

### FR Traceability

| FR | Covered By |
|---|---|
| FR-001 | AS-001, AS-002, AS-003, AS-004 |
| FR-002 | AS-001 |
| FR-003 | AS-001, AS-008, AS-009 |
| FR-004 | AS-001, AS-008, AS-009 |
| FR-005 | AS-001, AS-005 |
| FR-007 | AS-001, AS-005 |
| FR-008 | AS-001, AS-006, AS-007 |
| FR-009 | AS-001, AS-006 |
| FR-010 | AS-001, AS-010 |
| FR-011 | AS-001 (Python 3.11 — implicit in all scripts) |
| FR-012 | AS-001, AS-010 (local file storage) |
| FR-013 | AS-001 (logging — verified across all scenarios) |

---

## Phase 0: Research

*Purpose: Identify unknowns that must be resolved before design begins. Each question gets a recommendation and a decision status.*

### RQ-001: How does data flow between F6 scripts via the SKILL.md agent?

**Question**: Each F6 script is invoked individually by the SKILL.md agent. How does output from one script (e.g., the artifact registry from `build_registry.py`) get passed to the next script (e.g., `map_evidence.py`)?

| Option | Mechanism | Pros | Cons |
|---|---|---|---|
| A — stdout/stdin piping | Script prints JSON to stdout, agent captures it and passes to next script as stdin | Simple, no temp files, Unix-standard | Agent must support piping; large payloads may be unwieldy |
| B — Temp file handoff | Script writes JSON to a known temp file path, next script reads it | Simple, inspectable for debugging, no agent piping needed | Violates "no intermediate persistence" constraint |
| C — Agent memory | Script returns a Python dict, agent holds it in memory and passes as argument to next script | Cleanest, truly in-memory | Depends on how Cursor agent invokes Python scripts — may not support passing Python objects |

**Recommendation**: Option A (stdout/stdin) — simplest, no temp files, and the agent runtime likely captures stdout naturally. If the agent can't pipe between scripts, fall back to Option B with temp files in `.cursor/skills/rcsa/output/tmp/` (added to `.gitignore`).

**Decision**: ⏳ Needs team input — depends on how Cursor agent runtime actually invokes scripts.

### RQ-002: What is the exact citation format F5 will produce?

**Question**: `validate_citations.py` needs to parse citations from the LLM's output. What format will they be in?

| Format | Example | Regex Pattern |
|---|---|---|
| Bracket-ID | `[ART-001]` | `\[ART-\d{3}\]` |
| Bracket-ID with name | `[ART-001: scan_report.pdf]` | `\[ART-\d{3}:\s*.+?\]` |
| Parenthetical | `(see ART-001)` | `\(see\s+ART-\d{3}\)` |

**Recommendation**: Bracket-ID `[ART-001]` — simplest, unambiguous, easy to regex. This should be defined in the F6↔F5 contract and locked in F4's citation format spec.

**Decision**: ⏳ Must align with F4 and F5 — this is a cross-feature contract.

### RQ-003: What does the LLM response structure look like?

**Question**: `orchestrate_llm.py` needs to parse the LLM's response. What structure should F6 expect?

| Option | Structure | Parsing Complexity |
|---|---|---|
| A — Free-form Markdown | LLM writes narrative prose with headings per control | Medium — parse by heading, fragile if LLM changes format |
| B — Structured Markdown with delimiters | LLM wraps each control section in known delimiters (e.g., `<!-- CONTROL:AC -->...<!-- /CONTROL:AC -->`) | Low — reliable delimiter parsing |
| C — JSON response | LLM returns JSON with keys per control | Low — `json.loads()`, but LLM may produce invalid JSON |

**Recommendation**: Option B (structured Markdown with delimiters) — gives the LLM freedom to write natural prose while giving F6 reliable parsing anchors. F5's SKILL.md instructions would tell the LLM to wrap each section in delimiters.

**Decision**: ⏳ Must align with F5 — this is a cross-feature contract.

### RQ-004: How does F6 determine confidence tiers?

**Question**: The output includes confidence tiers per control. What determines the tier?

| Tier | Criteria (proposed) |
|---|---|
| **High** | ≥ 2 artifacts AND ≥ 1 test mapped, 0 hallucinated citations |
| **Medium** | ≥ 1 artifact OR ≥ 1 test mapped, 0 hallucinated citations |
| **Low** | Any evidence exists but hallucinated citations detected |
| **Gap** | No evidence mapped to this control |

**Recommendation**: Use the table above as the starting point. These thresholds are deterministic and enforceable in `map_evidence.py` (for gap/partial) and `validate_citations.py` (for hallucination downgrade).

**Decision**: ⏳ Needs team review — thresholds may need adjustment based on demo data.

### RQ-005: What are the exact F4 schema structures?

**Question**: `validate_input.py` validates against F4's JSON schemas. Are the schemas finalized?

**Recommendation**: F6 cannot begin implementation of `validate_input.py` until F4's schemas are locked. During development, F6 can use draft schemas and update when F4 finalizes. The F6↔F4 contract (Phase 1 deliverable) will document the exact schema versions F6 depends on.

**Decision**: ⏳ Blocked by F4 — schemas must be locked before F6 implementation begins. Not blocked for F6 planning or design.

### Research Summary

| RQ | Topic | Status | Blocking? |
|---|---|---|---|
| RQ-001 | Script-to-script data flow | ⏳ Needs team input | Blocks Phase 1 design |
| RQ-002 | Citation format | ⏳ Cross-feature contract | Blocks validate_citations.py design |
| RQ-003 | LLM response structure | ⏳ Cross-feature contract | Blocks orchestrate_llm.py design |
| RQ-004 | Confidence tier thresholds | ⏳ Needs team review | Blocks map_evidence.py design |
| RQ-005 | F4 schema structures | ⏳ Blocked by F4 | Blocks validate_input.py implementation (not design) |

**Phase 0 Gate**: All 5 research questions have recommendations. RQ-001 through RQ-004 need team decisions before Phase 1 design can begin. RQ-005 is an implementation dependency, not a design blocker.

---

*Sections remaining (to be completed after Phase 0 decisions):*

- *Risks and Mitigations*
- *Dependencies*
- *Phases and Milestones*
- *Design Decisions Log*
