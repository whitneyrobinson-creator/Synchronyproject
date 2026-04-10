# Project Specification: Synchrony Documentation Automation Skillset

**Project Repo**: `synchrony-doc-automation`
**Created**: 2026-03-24
**Status**: Draft
**Version**: 0.2

## Overview

Synchrony Documentation Automation Skillset is an agent skillset that turns technical repository artifacts — schemas, code files, test catalogs, and control libraries — into audit-ready compliance documentation, delivered as structured markdown reports with citations, confidence scores, and validation checks.

It is designed primarily for Synchrony's documentation teams who need to generate data dictionaries from database schemas, and secondarily for risk/compliance teams who need to produce evidence-backed control narratives from repository artifacts. The skillset is built on the Claude Agent Skills architecture (SKILL.md + scripts + assets) and is optimized for audit readiness and risk detection rather than speed alone.

---

## Repository Structure

This is the canonical layout of the repository when all three Data Dictionary features (F1, F2, F3) are fully built. Each feature plan shows only its own files — refer to this section for the complete picture.

```
synchrony-doc-automation/                    # Repository root
│
├── skills/
│   └── data-dictionary/
│       ├── SKILL.md                         # F1 — LLM instruction file
│       ├── scripts/                         # F2 — 6 Python pipeline scripts
│       │   ├── validate_input.py
│       │   ├── extract_fields.py
│       │   ├── attach_citations.py
│       │   ├── add_timestamps.py
│       │   ├── assemble_output.py
│       │   └── generate_qa_report.py
│       └── output/                          # F2 — created at runtime, NOT checked into repo
│           ├── data_dictionary.md           # Final deliverable
│           ├── qa_report.md                 # Final deliverable
│           └── intermediate/
│               ├── user_input.json          # Provided by user via Claude upload
│               ├── validated_schema.json
│               ├── extracted_fields.json
│               ├── extraction_warnings.json  # Conditional — only if warnings exist
│               ├── llm_output.json          # Written by F1 (LLM output)
│               ├── merged_fields.json
│               └── timestamped_fields.json
│
├── assets/                                  # F3 — static files, checked into repo
│   ├── data_dictionary_template.md
│   ├── qa_report_template.md
│   ├── sample_schema.json
│   ├── example_data_dictionary.md
│   ├── example_qa_report.md
│   └── glossary_template.json               # P3 placeholder — not required for demo day
│
└── specs/                                   # Planning docs — NOT runtime files
    ├── f1-skill.md-data-dictionary/
    │   ├── plan.md
    │   ├── research.md
    │   ├── data-model.md
    │   ├── quickstart.md
    │   ├── contracts/
    │   │   ├── input-contract.md
    │   │   └── output-contract.md
    │   └── tasks.md
    ├── f2-scripts-data-dictionary/
    │   ├── plan.md
    │   ├── research.md
    │   ├── data-model.md
    │   ├── quickstart.md
    │   ├── contracts.md
    │   └── tasks.md
    └── f3-assets-data-dictionary/
        ├── plan.md
        ├── research.md
        ├── data-model.md
        ├── quickstart.md
        ├── contracts.md
        └── tasks.md
```

**Notes:**
- `output/` and `output/intermediate/` are created at runtime by F2 scripts via `os.makedirs()`. They are not checked into the repository.
- `assets/` is at the repository root, not inside `skills/`.
- `specs/` contains planning documentation only. Nothing in `specs/` is read at runtime.

---

## User Scenarios & Testing *(mandatory)*

<!--
  User stories are PRIORITIZED as user journeys ordered by importance.
  Each story is INDEPENDENTLY TESTABLE — implementing just one still delivers a viable MVP.
  P1 is the most critical.
-->

### User Story 1 — Generate a Data Dictionary (Priority: P1)

A documentation team member provides a schema file in JSON format. The skillset processes the schema and returns a complete data dictionary containing field-level metadata — name, type, constraints, plain-language description, citation, confidence score, and last_verified timestamp — along with a QA report flagging any items needing clarification.

> **Note:** YAML and DDL input formats are documented as future extensions but are not supported for the May 7th demo.

**Why this priority**: This is one of two core end-to-end workflows required for the May 7th demo. A working P1 is a shippable MVP that demonstrates the full artifact-to-documentation pipeline.

**Independent Test**: Can be fully tested by providing a sample schema file and verifying the returned data dictionary contains all required fields for every column in the schema, plus a QA report with coverage stats.

**Acceptance Scenarios**:

1. **Given** a valid JSON schema file, **When** the user invokes the skill, **Then** the system returns a `data_dictionary.md` with all required fields: field_name, type, nullable, constraints/enums, description, source_path(s), evidence_refs, confidence, and last_verified.
2. **Given** a valid schema file, **When** the data dictionary is generated, **Then** every field description includes a citation to the source schema artifact.
3. **Given** valid inputs, **When** the skill completes, **Then** the system also produces a `qa_report.md` showing coverage stats and any [NEEDS CLARIFICATION] items.

---

### User Story 2 — Generate RCSA Control Narratives (Priority: P1)

A risk/compliance team member provides repository artifacts (artifact index, test catalog, and control library — all JSON). The skillset scans the evidence, maps it to four generic compliance controls, and returns citation-backed narratives for each control — explicitly flagging gaps where no evidence exists — along with a validation report showing citation resolution stats.

**Why this priority**: This is the second core end-to-end workflow required for the May 7th demo. It demonstrates the skillset's ability to assess audit readiness and flag risk, which is the primary value proposition for Synchrony.

**Independent Test**: Can be fully tested by providing sample artifact index + test catalog + control library and verifying the returned control narratives address all four controls with citations, gap flags where appropriate, and a validation report with resolution stats.

**Acceptance Scenarios**:

1. **Given** valid inputs (artifact index, test catalog, control library), **When** the user invokes the skill, **Then** the system returns `rcsa_control_narratives.md` with a summary table and per-control narratives (3–5 sentences each) with inline citations.
2. **Given** valid inputs where one control has no supporting evidence, **When** the skill completes, **Then** the system explicitly flags that control as a "Gap" — the system MUST NOT imply compliance without proof.
3. **Given** valid inputs, **When** the skill completes, **Then** every citation in the narratives resolves to a real artifact in the index.
4. **Given** valid inputs, **When** the skill completes, **Then** the system also produces a `validation_report.md` showing citation resolution stats (total, valid, invalid, coverage %).

---

### User Story 3 — Generate Pipeline Documentation (Priority: P2)

A documentation team member provides a codebase or pipeline definition. The skillset generates step-by-step documentation explaining inputs, outputs, transformations, and assumptions — with citations linking each documented step to the source code.

**Why this priority**: Validates the skillset's ability to extend beyond compliance documentation into general technical documentation. Independently demonstrable without needing the RCSA or data dictionary pipelines.

**Independent Test**: Can be fully tested by providing a sample codebase or pipeline definition and verifying the returned documentation covers all pipeline steps with citations to source code.

**Acceptance Scenarios**:

1. **Given** a valid codebase or pipeline definition, **When** the user invokes the skill, **Then** the system returns `pipeline_documentation.md` with step-by-step documentation covering inputs, outputs, transformations, and assumptions.
2. **Given** valid inputs, **When** the documentation is generated, **Then** every documented step includes a citation to the corresponding source code.

---

### User Story 4 — Generate Test Evidence Summary (Priority: P2)

A documentation team member provides test results (pass/fail, coverage stats). The skillset generates a summary of test execution results — total tests, passed, failed, coverage percentage — with links to corresponding test files and code under test.

**Why this priority**: Validates the skillset's ability to produce test-focused audit evidence. Uses similar inputs as the RCSA skill (test catalogs), demonstrating reuse of the artifact-to-doc pattern.

**Independent Test**: Can be fully tested by providing sample test results and verifying the returned summary accurately reflects pass/fail counts and links each result to its test file.

**Acceptance Scenarios**:

1. **Given** valid test results, **When** the user invokes the skill, **Then** the system returns `test_evidence_summary.md` with total tests, passed, failed, and coverage %.
2. **Given** valid inputs, **When** the summary is generated, **Then** every test result links to the corresponding test file and code under test.

---

### User Story 5 — DOCX Output Format (Priority: P3)

A user generates any document and requests DOCX format. The system produces the same content in DOCX in addition to Markdown.

**Why this priority**: A simple output format extension. Independently demonstrable in isolation, but represents the lowest complexity increment over Stories 1–4.

**Independent Test**: Can be fully tested by generating a data dictionary and requesting DOCX output, then verifying the DOCX contains identical content to the Markdown version.

**Acceptance Scenarios**:

1. **Given** a completed document generation, **When** the user requests DOCX format, **Then** the system produces a `.docx` file with identical content to the Markdown version.

---

### User Story 6 — Glossary Mapping (Priority: P3)

A documentation team member provides a glossary alongside a schema file. The system maps field names to consistent glossary labels in the data dictionary, ensuring organizational terminology is applied uniformly.

**Why this priority**: Validates the optional input layer for data dictionary generation. Independently demonstrable without changing the core pipeline.

**Independent Test**: Can be fully tested by providing a schema + glossary and verifying field names are mapped to their corresponding glossary labels in the output.

**Acceptance Scenarios**:

1. **Given** a schema file and a glossary, **When** the data dictionary is generated, **Then** field names are mapped to their corresponding glossary labels.

---

### User Story 7 — Change Detection (Priority: P4)

A documentation team member provides a current schema and a prior version. The system produces a `changes.md` showing added, removed, and modified fields with concise notes.

**Why this priority**: A nice-to-have diff feature. Independently demonstrable but not required for any other story.

**Independent Test**: Can be fully tested by providing a current and prior schema and verifying `changes.md` accurately reflects the differences.

**Acceptance Scenarios**:

1. **Given** a current schema and a prior version, **When** the skill completes, **Then** the system produces `changes.md` showing added, removed, and modified fields with concise notes.

---

### Edge Cases

- **Invalid schema format**: System rejects the input with a clear error message listing valid formats and allows the user to correct without restarting. For demo day, only JSON is supported; YAML and DDL return a clear message indicating they are planned for a future version.
- **Empty schema / no fields**: System returns an error explaining that the schema contains no extractable fields, rather than producing an empty data dictionary.
- **Missing optional inputs (glossary, profiling stats, prior version)**: System proceeds without them; output notes which optional inputs were not provided and which defaults were applied.
- **Control with no evidence**: System explicitly flags a "Gap" statement for that control — never implies compliance without proof.
- **All citations invalid**: System produces the validation report showing 0% resolution and flags the output as unreliable; does not suppress the report.
- **Schema with ambiguous or undocumented field names**: System generates best-effort descriptions and marks them as [NEEDS CLARIFICATION] with low confidence scores.
- **Extremely large schema (500+ fields)**: System processes all fields but may chunk the output; QA report notes total field count and processing completeness.
- **Duplicate field names in schema**: System flags duplicates in the QA report and includes both entries with a warning.

---

## Features *(mandatory)*

<!--
  Features are derived from the Claude Agent Skills architecture: each skill is composed of
  SKILL.md (LLM instructions), Scripts (deterministic code), and Assets (templates and static resources).
  The two skills are independent (no runtime dependency), so no integration feature is required.

  Build Order: SKILL.md → Scripts → Assets (per skill). SKILL.md is built first to understand
  what the LLM needs to do, then Scripts to support it with code, then Assets for templates.
-->

### Skill 1 — Data Dictionary Generation

| Feature | Name | Responsibility | Simple Version | Functional Requirements |
|---|---|---|---|---|
| **F1** | SKILL.md — Data Dictionary | Descriptions, confidence scores, clarification flags, glossary interpretation | The LLM thinks | FR-007, FR-008, FR-009, FR-011 |
| **F2** | Scripts — Data Dictionary | Parsing, extracting, validating, citing, timestamping, counting, assembling | The code works | FR-001, FR-005, FR-006, FR-008, FR-009, FR-010, FR-011, FR-012, FR-013 |
| **F3** | Assets — Data Dictionary | Templates that define what the outputs look like | The blank forms | FR-012, FR-013 |

### Skill 2 — RCSA Control Narrative Generation

| Feature | Name | Description | Functional Requirements |
|---|---|---|---|
| **F4** | SKILL.md — RCSA | Step-by-step instructions for the agent: input validation, artifact index parsing, test catalog parsing, control library parsing, evidence-to-control mapping, narrative generation with inline citations, gap flagging, citation validation, summary table generation, and validation report generation. References the validated templates from F6. | FR-002, FR-003, FR-004, FR-005, FR-015, FR-016, FR-017, FR-018, FR-019, FR-020 |
| **F5** | Scripts — RCSA | Deterministic Python scripts that parse all three JSON input files, build an artifact registry, map evidence to the four compliance controls, validate every citation against the registry, generate the summary table, populate templates, and produce the validation report with resolution stats. | FR-002, FR-003, FR-004, FR-005, FR-015, FR-016, FR-019, FR-020, FR-021, FR-022 |
| **F6** | Assets — RCSA | Completed example RCSA control narratives and validation report that validate the output format is correct and audit-ready. Blank templates derived from the validated examples for script population. Sample artifact index, test catalog, and control library files for testing. | FR-021, FR-022 |

### Build Order

| Order | Feature | Rationale |
|---|---|---|
| 1 | F1 — SKILL.md: Data Dictionary | Understand what the LLM needs to do first |
| 2 | F2 — Scripts: Data Dictionary | Build the code that supports the LLM work |
| 3 | F3 — Assets: Data Dictionary | Create the templates that define output format (completes Skill 1) |
| 4 | F4 — SKILL.md: RCSA | Understand what the LLM needs to do first |
| 5 | F5 — Scripts: RCSA | Build the code that supports the LLM work |
| 6 | F6 — Assets: RCSA | Create the templates that define output format (completes Skill 2) |

---

## Requirements *(mandatory)*

### Functional Requirements

| FR | Description | Owner |
|---|---|---|
| **FR-001** | System MUST accept schema extracts in JSON format as input for data dictionary generation. YAML and DDL input support is documented as a future extension. | Script |
| **FR-002** | System MUST accept artifact indexes (JSON) listing all discovered files, functions, and classes in a repository | Script |
| **FR-003** | System MUST accept test catalogs (JSON) listing test names, file paths, and pass/fail status | Script |
| **FR-004** | System MUST accept a control library (JSON) defining controls, objectives, and expected evidence types | Script |
| **FR-005** | All input validation — file format, schema structure — MUST be handled by deterministic script logic before any LLM processing begins | Script |
| **FR-006** | System MUST extract all fields from a schema (field name, type, nullable, constraints/enums) | Script |
| **FR-007** | System MUST generate a plain-language description (≤25 words) for each field | LLM |
| **FR-008** | System MUST include a citation (source path, evidence refs) for every field description | Script + LLM |
| **FR-009** | System MUST assign a confidence score to each field description | Script + LLM |
| **FR-010** | System MUST include a last_verified timestamp for each field | Script |
| **FR-011** | If a glossary is provided, system MUST map field names to consistent glossary labels. Glossary mapping is a P3 feature — not included for demo day. | Script + LLM |
| **FR-012** | System MUST produce `data_dictionary.md` as the primary output; structure and formatting enforced by a defined output template. CSV output format is documented as a future extension. | Script |
| **FR-013** | System MUST produce `qa_report.md` showing coverage stats, confidence distribution breakdown (High, Medium, Low, N/A counts with percentages), and flagged items | Script |
| **FR-014** | If a prior version is provided, system MAY produce `changes.md` showing added/removed/modified fields | Script |
| **FR-015** | System MUST scan a repository and build an index of all artifacts (code files, tests, configs) | Script |
| **FR-016** | System MUST map discovered evidence to defined controls (Access Control, Change Management, Data Quality, Incident Handling) | Script + LLM |
| **FR-017** | System MUST generate auditor-friendly narratives (3–5 sentences per control) with inline citations | LLM |
| **FR-018** | System MUST explicitly flag a "Gap" statement when no evidence exists for a control — system MUST NOT imply compliance without proof | LLM |
| **FR-019** | System MUST validate that every citation resolves to a real artifact in the index | Script |
| **FR-020** | System MUST produce a summary table showing which controls have evidence vs. gaps | Script |
| **FR-021** | System MUST produce `rcsa_control_narratives.md` as the primary output; structure and formatting enforced by a defined output template | Script |
| **FR-022** | System MUST produce `validation_report.md` showing citation resolution stats (total, valid, invalid, coverage %) | Script |
| **FR-023** | System MUST accept a codebase or pipeline definition as input | Script |
| **FR-024** | System MUST generate step-by-step documentation explaining inputs, outputs, transformations, and assumptions of a pipeline | LLM |
| **FR-025** | System MUST include citations linking each documented step to the source code | Script + LLM |
| **FR-026** | System MUST produce `pipeline_documentation.md` as the primary output; structure and formatting enforced by a defined output template | Script |
| **FR-027** | System MUST accept test results (pass/fail, coverage stats) as input | Script |
| **FR-028** | System MUST generate a summary of test execution results (total tests, passed, failed, coverage %) | Script + LLM |
| **FR-029** | System MUST link each test result to the corresponding test file and code under test | Script |
| **FR-030** | System MUST produce `test_evidence_summary.md` as the primary output; structure and formatting enforced by a defined output template | Script |
| **FR-031** | All primary outputs MUST be in Markdown format | Script |
| **FR-032** | System MAY support optional DOCX output format | Script |

---

### Key Entities *(data and concepts central to the project)*

#### P1 — Demo Day (Stories 1 & 2)

- **Schema Extract** — A JSON file containing database/table field definitions (column names, types, constraints, nullability). Primary input for data dictionary generation. JSON format is required for demo day; YAML and DDL support is documented as a future extension.

- **Artifact Index** — A JSON file listing all discovered files, functions, and classes in a repository. Primary input for RCSA control narrative generation.

- **Test Catalog** — A JSON file listing test names, file paths, and pass/fail status. Input for RCSA control narrative generation and test evidence summary.

- **Control Library** — A JSON file defining the four generic compliance controls (Access Control, Change Management, Data Quality, Incident Handling), their objectives, and expected evidence types. Input for RCSA control narrative generation.

- **Data Dictionary** — The generated Markdown document containing field-level metadata: field_name, type, nullable, constraints/enums, description (≤25 words), source_path(s), evidence_refs, confidence, and last_verified. Primary output of Skill 1. (Note: `glossary_label` is a P3 feature and is not included in the demo day output.)

- **Control Narrative** — An auditor-friendly 3–5 sentence paragraph per compliance control, with inline citations linking every claim to a specific repository artifact. Explicitly flags gaps where no evidence exists. Primary output of Skill 2.

- **QA Report** — A generated report showing coverage statistics, confidence distribution breakdown, citation checks, and a list of [NEEDS CLARIFICATION] items. Produced alongside the data dictionary.

- **Validation Report** — A generated report showing citation resolution statistics: total citations, valid, invalid, and coverage percentage. Produced alongside the RCSA control narratives.

#### P2 — Documented for Future (Stories 3 & 4)

- **Pipeline Definition** — A codebase or pipeline configuration file describing data flow steps, transformations, inputs, and outputs. Primary input for pipeline documentation generation.

- **Test Results** — Test execution results including pass/fail status and coverage statistics. Primary input for test evidence summary generation.

- **Pipeline Documentation** — The generated step-by-step Markdown document explaining inputs, outputs, transformations, and assumptions of a data pipeline, with citations to source code.

- **Test Evidence Summary** — The generated Markdown summary of test execution results (total, passed, failed, coverage %) with links to corresponding test files and code under test.

#### P3 — Future (Story 5)

- **Doc Pack** — The complete bundle of all generated documents from a single repository input. Combines data dictionary, control narratives, pipeline documentation, and test evidence summary into one deliverable.

---

## Success Criteria *(mandatory)*

<!--
  All criteria are technology-agnostic and measurable.
  "Audit-ready" rubric: citations resolve to real artifacts, no fabricated content,
  gaps explicitly flagged, template fully populated, confidence scores assigned.
-->

### Measurable Outcomes — Demo Day (P1)

| SC | Outcome | Theme |
|---|---|---|
| **SC-001** | Time to generate a data dictionary is reduced from 60–120 min (manual) to ~5–10 min per repo using the skillset | Effort Reduction |
| **SC-002** | ≥ 95% of fields in the source schema are correctly extracted and present in the generated data dictionary | Output Quality |
| **SC-003** | ≥ 90% of field descriptions in the data dictionary include a citation to the source schema artifact | Traceability |
| **SC-004** | ≥ 90% of template sections are populated in every generated data dictionary | Output Quality |
| **SC-005** | ≥ 85% of true compliance gaps (controls with missing/weak evidence) are correctly flagged in the RCSA control narratives | Risk Detection |
| **SC-006** | ≤ 5% of controls or tests referenced in RCSA narratives are invented (not present in the source artifacts) | Output Quality |
| **SC-007** | ≥ 90% of citations in RCSA control narratives resolve to real artifacts in the repository index | Traceability |
| **SC-008** | ≥ 3 out of 4 compliance controls are addressed (with evidence or explicit gap flag) in every RCSA run | Output Quality |
| **SC-009** | ≥ 90% structural match when the same skill is run on the same inputs multiple times | Processing Reliability |
| **SC-010** | AI-generated documentation scores ≥ 80% on an evaluation rubric when compared against human-written ground truth documentation | Output Quality |

### Measurable Outcomes — Future (P2/P3, Not Measured for Demo Day)

> **Note:** The following success criteria apply to features documented as future extensions (Pipeline Documentation, Test Evidence Summary, Doc Pack). They will be measured when those features are built.

| SC | Outcome | Theme |
|---|---|---|
| **SC-011** | ≥ 90% of pipeline steps are documented with inputs, outputs, and transformations in the generated pipeline documentation | Output Quality |
| **SC-012** | ≥ 90% of documented pipeline steps include a citation to the corresponding source code | Traceability |
| **SC-013** | ≥ 95% of pass/fail counts in the test evidence summary match the actual test results provided as input | Output Quality |
| **SC-014** | ≥ 90% of test results in the test evidence summary are linked to their corresponding test file and code under test | Traceability |
| **SC-015** | ≥ 90% of applicable document types are successfully generated when the Doc Pack is invoked for a repository | Processing Reliability |

