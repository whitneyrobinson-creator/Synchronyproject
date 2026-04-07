# Feature Specification: F4 — RCSA Assets (Templates, Sample Data, and Control Library)

**Feature Branch**: `f4-rcsa-assets`  
**Created**: 2026-04-07  
**Status**: Draft  
**Input**: User description: "Create the assets package for the RCSA Control Narrative Generation skill, including output templates, sample input data, and the control library that defines compliance controls and their expected evidence types. F4 is the final feature built — it formalizes the output structures defined by F5 (SKILL.md) and the pipeline requirements established by F6 (Scripts) into reusable, versioned assets."

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 — Provide Output Templates for Consistent Formatting (Priority: P1)

A risk/compliance team member expects the skill to produce consistently formatted audit-ready documentation every time it runs. The assets package provides Markdown output templates that formalize the output structure already defined inline by F5 (SKILL.md) and enforced by F6 (Scripts). The narrative output template defines the exact structure of `rcsa_control_narratives.md` (summary table, per-control narrative sections, gap flags) and the validation report template defines the structure of `validation_report.md` (citation resolution statistics). These templates replace the inline definitions from F5 and become the single source of truth for output formatting.

**Why this priority**: Without formalized output templates, the output structure exists only inline in the SKILL.md and scripts — making it harder to maintain, version, and audit. Templates ensure consistency across runs and provide a clear contract between F5, F6, and the final output. Required for the May 7th demo.

**Independent Test**: Can be fully tested by verifying that the narrative output template contains all required sections (summary table, four control sections with narrative/gap placeholders, citation format) and that the validation report template contains all required fields (total citations, valid, invalid, coverage percentage). Templates must match the output structure already produced by F5 and F6.

**Acceptance Scenarios**:

1. **Given** the narrative output template exists, **When** a reviewer inspects it, **Then** it contains a summary table section, four control narrative sections (AC, CM, DQ, IH), inline citation format examples, and gap flag format.
2. **Given** the validation report template exists, **When** a reviewer inspects it, **Then** it contains fields for total citations, valid citations, invalid citations, and coverage percentage.
3. **Given** both templates are available, **When** they are compared to the output currently produced by F5 (SKILL.md) and F6 (Scripts), **Then** the templates match the existing output structure — no breaking changes.
4. **Given** both templates are available, **When** the SKILL.md (F5) and Scripts (F6) are updated to reference them, **Then** the templates are accessible at the expected file paths within the assets directory.

---

### User Story 2 — Provide Sample Input Data for Testing and Demo (Priority: P1)

A developer or team member needs sample input files to test the skill pipeline end-to-end without requiring a real repository. The assets package formalizes the sample data that was created manually during F5 and F6 development into versioned, reusable JSON files: a sample artifact index, a sample test catalog, and a sample control library — all representing a realistic but small demo-scale repository.

**Why this priority**: During F5 and F6 development, sample data was created ad hoc for testing. F4 formalizes this into a canonical set of sample inputs that anyone can use to run the demo or test the pipeline. Required for the May 7th demo.

**Independent Test**: Can be fully tested by verifying that each sample input file is valid JSON, follows the expected schema (as defined by F6's input validator), and contains enough data to exercise all four compliance controls (some with evidence, some without — to test both narrative generation and gap detection).

**Acceptance Scenarios**:

1. **Given** the sample artifact index exists, **When** it is parsed, **Then** it is valid JSON containing file entries with identifiers, file paths, and artifact types.
2. **Given** the sample test catalog exists, **When** it is parsed, **Then** it is valid JSON containing test entries with test names, file paths, and pass/fail status.
3. **Given** the sample control library exists, **When** it is parsed, **Then** it is valid JSON containing all four controls (AC, CM, DQ, IH) with objectives and expected evidence types.
4. **Given** all three sample inputs are provided to the F6 pipeline, **When** the pipeline runs, **Then** at least one control has sufficient evidence (for narrative generation) and at least one control lacks evidence (for gap detection testing).
5. **Given** the sample inputs, **When** they are validated by F6's input validation script, **Then** they pass all validation checks (existence, JSON validity, schema conformance) with 0 errors.

---

### User Story 3 — Define the Control Library as Source of Truth (Priority: P1)

A risk/compliance team member needs a single authoritative definition of what each compliance control requires. The control library JSON file defines the four controls (Access Control, Change Management, Data Quality, Incident Handling), their objectives, and the expected evidence types for each. F6 (Scripts) uses this for evidence mapping, and F5 (SKILL.md) uses it to understand what constitutes sufficient evidence. The control library was referenced by F5 and F6 during development; F4 formalizes it as a versioned, canonical asset.

**Why this priority**: The control library is the source of truth for evidence mapping. Without a formalized version, the control definitions exist only in scattered references across F5 and F6. Required for the May 7th demo.

**Independent Test**: Can be fully tested by verifying the control library contains all four controls with objectives and expected evidence types, and that the evidence types are specific enough to support deterministic mapping by F6 (Scripts).

**Acceptance Scenarios**:

1. **Given** the control library exists, **When** it is inspected, **Then** it defines exactly four controls: Access Control, Change Management, Data Quality, and Incident Handling.
2. **Given** each control in the library, **When** its definition is reviewed, **Then** it includes a control objective and a list of expected evidence types.
3. **Given** the expected evidence types for each control, **When** they are compared to the sample artifact index, **Then** the evidence types are specific enough to support deterministic matching (not vague categories).
4. **Given** the control library, **When** it is used by F6's evidence mapper, **Then** it produces the same mapping results as during F6 development — no regressions.

---

### User Story 4 — Provide Citation Format Specification (Priority: P2)

The assets package defines the exact citation format that F5 (SKILL.md) instructs the LLM to use and that F6 (Scripts) validates against. This formalizes the citation format that was defined inline during F5 and F6 development into a single reusable reference document.

**Why this priority**: Citation consistency is important for audit traceability, but the skill can still function with minor citation format variations. Prioritized after the core templates, sample data, and control library.

**Independent Test**: Can be fully tested by verifying the citation format specification defines the syntax, provides examples, and matches the format already used by F5 and validated by F6.

**Acceptance Scenarios**:

1. **Given** the citation format specification exists, **When** a reviewer inspects it, **Then** it defines the exact syntax for inline citations (e.g., `[artifact-id: file-path]`).
2. **Given** the citation format, **When** example citations are compared to the sample artifact index, **Then** the citation identifiers match real entries in the index.
3. **Given** the citation format, **When** it is compared to the format F6's citation validator already checks against, **Then** they are identical — no discrepancies.

---

### Edge Cases

- What happens when the control library is missing a control? The system should fail validation and report which control is missing rather than proceeding with incomplete data.
- What happens when sample input files have malformed JSON? F6's input validation catches this before processing begins — F4 must ensure sample data passes F6 validation.
- What happens when the control library defines evidence types that don't match any artifacts in the sample data? This is expected for gap detection testing — at least one control should intentionally lack matching evidence.
- What happens when output templates are modified after F5 and F6 are built? Templates are versioned; changes require re-testing the full pipeline to ensure no regressions.
- What happens when the citation format specification conflicts with what F6's citation validator expects? The citation format spec must match F6's existing validation logic — F4 formalizes, it does not redefine.
- What happens when a new control needs to be added? The control library, templates, SKILL.md (F5), and scripts (F6) all need updating — document this as a known maintenance requirement.

---

## Requirements *(mandatory)*

### Functional Requirements

| ID | Requirement |
|----|-------------|
| **FR-001** | Assets MUST include a narrative output template (`rcsa_control_narratives_template.md`) that formalizes the output structure defined by F5 (SKILL.md) |
| **FR-002** | Assets MUST include a validation report template (`validation_report_template.md`) that formalizes the validation output structure from F6 (Scripts) |
| **FR-003** | Assets MUST include a sample artifact index (`sample_artifact_index.json`) with realistic demo-scale data |
| **FR-004** | Assets MUST include a sample test catalog (`sample_test_catalog.json`) with realistic demo-scale data |
| **FR-005** | Assets MUST include a control library (`control_library.json`) defining all four compliance controls with objectives and expected evidence types |
| **FR-006** | The control library MUST define exactly four controls: Access Control, Change Management, Data Quality, and Incident Handling |
| **FR-007** | All sample JSON files MUST be valid JSON, follow a consistent schema, and pass F6's input validation with 0 errors |
| **FR-008** | Sample data MUST include scenarios that exercise both narrative generation (sufficient evidence) and gap detection (missing evidence) |
| **FR-009** | Assets MUST include a citation format specification defining the exact syntax for inline citations, matching F6's citation validator |
| **FR-010** | All asset files MUST be accessible at documented file paths within the `.cursor/skills/rcsa/assets/` directory |
| **FR-011** | Output templates MUST match the output structure already produced by F5 and F6 — no breaking changes |
| **FR-012** | All asset files MUST use UTF-8 encoding |

### Key Entities

| Entity | Description |
|--------|-------------|
| **Narrative Output Template** | Markdown template formalizing the structure of `rcsa_control_narratives.md` — includes summary table, per-control sections, citation format, and gap flag format. Based on the inline structure defined by F5. |
| **Validation Report Template** | Markdown template formalizing the structure of `validation_report.md` — includes total citations, valid, invalid, and coverage percentage fields. Based on the output structure from F6. |
| **Sample Artifact Index** | JSON file listing sample repository files, functions, and classes with identifiers and types. Must pass F6's input validation. |
| **Sample Test Catalog** | JSON file listing sample test names, file paths, and pass/fail status. Must pass F6's input validation. |
| **Control Library** | JSON file defining the four compliance controls, their objectives, and expected evidence types. Source of truth for evidence mapping in F6. |
| **Citation Format Specification** | Document defining the exact syntax for inline citations used in narratives (F5) and validated by scripts (F6). |

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

| ID | Criteria |
|----|----------|
| **SC-001** | All asset files are valid and parseable — 0 JSON parsing errors across all sample input files |
| **SC-002** | Output templates contain all required sections as defined in the spec's acceptance scenarios and match F5/F6 output structure |
| **SC-003** | Sample data exercises all four controls — at least 2 controls with sufficient evidence and at least 1 control with intentionally missing evidence |
| **SC-004** | Control library defines exactly 4 controls with objectives and ≥ 2 expected evidence types per control |
| **SC-005** | Full pipeline (F5 + F6 + F4 assets) can run end-to-end using only the sample data provided in assets — no external data required for demo |
| **SC-006** | All sample input files pass F6's input validation script with 0 errors — confirming backward compatibility |

---

## Assumptions

- F5 (SKILL.md) and F6 (Scripts) are built before F4. F4 formalizes the output structures and sample data that were defined inline during F5 and F6 development.
- The four compliance controls (AC, CM, DQ, IH) are fixed for the scope of this project. Adding new controls would require updating the control library, templates, SKILL.md, and scripts.
- Sample data represents a small, demo-scale repository. It is not intended to represent enterprise-scale inputs.
- Output templates formalize existing structure — they do not introduce new formatting or sections that F5 and F6 don't already support.
- The control library's expected evidence types are specific enough for deterministic matching by F6 (Scripts). Vague categories (e.g., "security-related files") are not acceptable.
- Asset files are static for the duration of the demo. They may be updated between development iterations but are not modified at runtime.
- All asset files use UTF-8 encoding.
- The citation format specification must match what F6's citation validator already checks — F4 documents the format, it does not redefine it.
