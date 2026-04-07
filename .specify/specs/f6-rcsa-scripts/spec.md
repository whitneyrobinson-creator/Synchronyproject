# Feature Specification: F6 — RCSA Scripts (Deterministic Python Pipeline)

**Feature Branch**: `f6-rcsa-scripts`  
**Created**: 2026-04-07  
**Status**: Draft  
**Input**: User description: "Build the deterministic Python scripts that support the RCSA Control Narrative Generation skill — handling input validation, artifact registry building, evidence-to-control mapping, LLM invocation orchestration, citation validation, and output file writing."

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Validate Input Files Before Processing (Priority: P1)

A risk/compliance team member provides repository input files (artifact index, test catalog) to the skill. Before any LLM processing occurs, the scripts validate that all input files exist, are valid JSON, and conform to the expected schema. If validation fails, the scripts return a clear error message and do not invoke the LLM.

**Why this priority**: Input validation is the first gate in the pipeline. If malformed data reaches the LLM, the output will be unreliable. This supports Constitution Principle 2 (Graceful Degradation — scripts must function without LLM). Required for the May 7th demo.

**Independent Test**: Can be fully tested by providing valid and invalid input files and verifying the scripts accept valid inputs and reject invalid ones with clear error messages — no LLM required.

**Acceptance Scenarios**:

1. **Given** all input files exist and are valid JSON with correct schema, **When** the validation script runs, **Then** it returns a success status and proceeds to the next pipeline step.
2. **Given** an input file is missing, **When** the validation script runs, **Then** it returns an error message identifying the missing file and does NOT proceed.
3. **Given** an input file contains malformed JSON, **When** the validation script runs, **Then** it returns an error message identifying the parsing error and does NOT proceed.
4. **Given** an input file has valid JSON but incorrect schema (e.g., missing required fields), **When** the validation script runs, **Then** it returns an error message identifying the schema violation and does NOT proceed.

---

### User Story 2 - Build Artifact Registry and Map Evidence to Controls (Priority: P1)

After input validation passes, the scripts build an artifact registry (a structured index of all repository files, functions, and classes) and then map each artifact to the appropriate compliance control(s) based on the control library definitions. The mapped evidence is the structured input that the LLM (F5 — SKILL.md) receives.

**Why this priority**: Evidence mapping is the core deterministic logic that feeds the LLM. Without it, the LLM has no structured input to reason over. This is the bridge between raw repository data and LLM-ready evidence. Required for the May 7th demo.

**Independent Test**: Can be fully tested by providing sample input files and verifying the scripts produce a correctly structured mapped evidence output — no LLM required.

**Acceptance Scenarios**:

1. **Given** valid input files and a control library, **When** the evidence mapping script runs, **Then** it produces a mapped evidence JSON structure with artifacts assigned to the correct controls.
2. **Given** artifacts that match multiple controls, **When** the mapping script runs, **Then** the artifact appears under each matching control.
3. **Given** a control with no matching artifacts in the input, **When** the mapping script runs, **Then** that control's evidence list is empty (enabling the LLM to flag a Gap).
4. **Given** the mapped evidence output, **When** it is inspected, **Then** each entry includes the artifact identifier, file path, artifact type, and the control it maps to.

---

### User Story 3 - Orchestrate LLM Invocation (Priority: P1)

The scripts package the mapped evidence into the correct format, invoke the LLM with the SKILL.md instructions, handle the LLM response, and manage errors (timeouts, rate limits, malformed responses). If the LLM fails, the scripts produce a graceful degradation output rather than crashing.

**Why this priority**: LLM orchestration is the integration point between the deterministic pipeline and the AI reasoning step. Without it, the SKILL.md cannot be executed. Graceful degradation is required by Constitution Principle 2. Required for the May 7th demo.

**Independent Test**: Can be tested by mocking LLM responses (success, timeout, rate limit, malformed) and verifying the scripts handle each case correctly.

**Acceptance Scenarios**:

1. **Given** valid mapped evidence, **When** the orchestration script invokes the LLM, **Then** it passes the mapped evidence in the format expected by the SKILL.md and receives a response.
2. **Given** the LLM returns a valid response, **When** the orchestration script processes it, **Then** it passes the response to the citation validation step.
3. **Given** the LLM times out or returns a rate limit error, **When** the orchestration script handles the error, **Then** it retries with exponential backoff (up to 3 retries) and logs the error.
4. **Given** the LLM fails after all retries, **When** the orchestration script handles the failure, **Then** it produces a graceful degradation output (e.g., "LLM unavailable — raw evidence provided without narratives") and does NOT crash.
5. **Given** the LLM returns a malformed response, **When** the orchestration script validates the response format, **Then** it flags the issue and falls back to graceful degradation.

---

### User Story 4 - Validate Citations in LLM Output (Priority: P1)

After the LLM generates narratives, the scripts validate that every inline citation references a real artifact from the artifact registry. Citations that reference non-existent artifacts are flagged as invalid. The validation results are included in the validation report.

**Why this priority**: Citation validation is the primary defense against LLM hallucination. Without it, the audit documentation could reference artifacts that don't exist. This directly supports Constitution Principle 1 (Accuracy Over Speed). Required for the May 7th demo.

**Independent Test**: Can be fully tested by providing LLM output with a mix of valid and invalid citations and verifying the scripts correctly identify which are real and which are hallucinated — can use mocked LLM output.

**Acceptance Scenarios**:

1. **Given** LLM output with inline citations, **When** the citation validation script runs, **Then** it checks each citation against the artifact registry.
2. **Given** a citation that references a real artifact in the registry, **When** it is validated, **Then** it is marked as valid.
3. **Given** a citation that references an artifact NOT in the registry (hallucinated), **When** it is validated, **Then** it is marked as invalid and flagged in the validation report.
4. **Given** the validation results, **When** the validation report is generated, **Then** it includes total citations, valid count, invalid count, and a list of invalid citations with details.

---

### User Story 5 - Write Output Files (Priority: P2)

The scripts write the final output files to the local filesystem: `rcsa_control_narratives.md` (the narrative document) and `validation_report.md` (the citation validation results). Output follows the structure defined by F5's inline output definitions (later formalized into templates by F4).

**Why this priority**: File writing is the final step that produces the deliverable artifacts. The skill still provides value through console output even without file writing, but file output is needed for the demo. Required for the May 7th demo.

**Independent Test**: Can be fully tested by providing processed narrative content and verifying the scripts write correctly formatted Markdown files to the expected paths.

**Acceptance Scenarios**:

1. **Given** validated narrative content, **When** the output writer runs, **Then** it creates `rcsa_control_narratives.md` at the expected output path.
2. **Given** citation validation results, **When** the output writer runs, **Then** it creates `validation_report.md` at the expected output path.
3. **Given** both output files are written, **When** they are inspected, **Then** they follow the output structure defined by F5 (summary table, per-control sections, gap flags, citation format).
4. **Given** the output directory does not exist, **When** the output writer runs, **Then** it creates the directory before writing files.

---

### Edge Cases

- What happens when the artifact index is extremely large (>10MB)? The scripts should log a warning but attempt to process. Document as a known limitation for demo-scale only.
- What happens when the control library defines a control not in the expected four? The scripts should reject it with a clear error — only AC, CM, DQ, IH are supported.
- What happens when the LLM returns an empty response? The scripts should treat this as a failure and fall back to graceful degradation.
- What happens when the LLM returns narratives but no citations? The citation validation step should flag 0 valid citations and the validation report should reflect this.
- What happens when two artifacts have the same identifier? The scripts should detect the duplicate during registry building and raise a validation error.
- What happens when the output directory is read-only? The scripts should catch the permission error and report it clearly rather than crashing.
- What happens when the LLM rate limit is exceeded? Retry with exponential backoff, up to 3 attempts, then graceful degradation.

---

## Requirements *(mandatory)*

### Functional Requirements

| ID | Requirement |
|----|-------------|
| **FR-001** | Scripts MUST validate all input files (existence, JSON validity, schema conformance) before any processing begins |
| **FR-002** | Scripts MUST build an artifact registry from the input artifact index, indexing files by identifier, path, and type |
| **FR-003** | Scripts MUST map artifacts to compliance controls using the control library's expected evidence types |
| **FR-004** | Scripts MUST produce a structured mapped evidence output (JSON) that the LLM can consume |
| **FR-005** | Scripts MUST orchestrate LLM invocation, passing mapped evidence in the format expected by the SKILL.md |
| **FR-006** | Scripts MUST implement retry logic with exponential backoff for LLM rate limits and timeouts (up to 3 retries) |
| **FR-007** | Scripts MUST handle LLM failures gracefully — producing degradation output instead of crashing (Constitution Principle 2) |
| **FR-008** | Scripts MUST validate every inline citation in LLM output against the artifact registry |
| **FR-009** | Scripts MUST generate a validation report with total citations, valid count, invalid count, and invalid citation details |
| **FR-010** | Scripts MUST write output files (`rcsa_control_narratives.md`, `validation_report.md`) to the local filesystem as Markdown |
| **FR-011** | Scripts MUST be written in Python 3.11 (Constitution Tech Stack requirement) |
| **FR-012** | Scripts MUST use only local file storage — no database, no cloud storage (Constitution constraint) |
| **FR-013** | Scripts MUST log all processing steps and errors for debugging and audit trail purposes |
| **FR-014** | Scripts MUST store any API keys in environment variables (`.env` file) — never hardcoded in source files |

### Key Entities

| Entity | Description |
|--------|-------------|
| **Input Validator** | Script/module that checks input files for existence, JSON validity, and schema conformance. First step in the pipeline. |
| **Artifact Registry** | In-memory structured index of all repository artifacts (files, functions, classes) built from the input artifact index. |
| **Evidence Mapper** | Script/module that maps artifacts to compliance controls based on the control library's expected evidence types. |
| **Mapped Evidence** | Structured JSON output of the evidence mapper — the input that the LLM receives. Contains artifacts grouped by control. |
| **LLM Orchestrator** | Script/module that packages mapped evidence, invokes the LLM, handles responses, and manages errors (retries, graceful degradation). |
| **Citation Validator** | Script/module that checks every inline citation in LLM output against the artifact registry to detect hallucinations. |
| **Output Writer** | Script/module that writes the final Markdown output files to the local filesystem. |
| **Validation Report** | Markdown file containing citation validation statistics and details of any invalid (hallucinated) citations. |

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

| ID | Criteria |
|----|----------|
| **SC-001** | Input validation catches 100% of malformed JSON and missing required fields — 0 invalid inputs reach the LLM |
| **SC-002** | Evidence mapping correctly assigns ≥ 90% of artifacts to their appropriate controls (measured against manually verified test data) |
| **SC-003** | Citation validation detects ≥ 95% of hallucinated citations (citations referencing non-existent artifacts) |
| **SC-004** | Graceful degradation activates on 100% of LLM failures — the pipeline never crashes due to LLM unavailability |
| **SC-005** | Full pipeline (validate → build registry → map evidence → invoke LLM → validate citations → write output) completes in < 60 seconds for demo-scale inputs |
| **SC-006** | All API keys are stored in environment variables — 0 hardcoded secrets in source files |

---

## Assumptions

- F5 (SKILL.md) is built first and defines the expected output structure. The scripts are built to support what F5 establishes — packaging inputs for the LLM and post-processing its output.
- The control library defines exactly four controls (AC, CM, DQ, IH). Adding new controls would require updating the evidence mapper.
- Input files are JSON format. The scripts do not parse other formats (CSV, XML, etc.).
- The LLM is Claude Sonnet, accessed through the Claude Agent Skills runtime environment. The orchestrator is designed for this specific integration.
- Sample input data for testing is created manually during F6 development. F4 (Assets) will later formalize these into reusable sample data files.
- The scripts run locally on the developer's machine. No cloud deployment or CI/CD pipeline is required for the demo.
- Output files are written to a local directory. The scripts do not upload or transmit output files.
- All scripts use UTF-8 encoding for file I/O.
