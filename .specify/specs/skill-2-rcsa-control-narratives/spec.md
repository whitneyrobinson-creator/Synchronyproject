# Feature Specification: Skill 2 — RCSA Control Narrative Generation

**Feature Branch**: `skill-2-rcsa-control-narratives`
**Created**: 2026-04-06
**Status**: Draft
**Input**: User provides artifact index (JSON), test catalog (JSON), and control library (JSON). The skill processes these inputs to generate citation-backed control narratives and a validation report.

## User Scenarios & Testing *(mandatory)*

### User Story 1 — Generate Evidence-Backed Control Narratives (Priority: P1)

A risk/compliance team member provides repository artifacts (artifact index, test catalog, and control library — all JSON). The skill maps evidence to four compliance controls — Access Control (AC), Change Management (CM), Data Quality (DQ), and Incident Handling (IH) — and generates 3–5 sentence auditor-friendly narratives per control with inline citations referencing real artifacts.

**Why this priority**: This is the core end-to-end workflow for Skill 2, required for the May 7th demo. It demonstrates the skillset's ability to turn repository evidence into audit-ready compliance documentation — the primary value proposition for Synchrony.

**Independent Test**: Can be fully tested by providing a sample artifact index + test catalog + control library and verifying the returned `rcsa_control_narratives.md` contains a narrative for each of the four controls with inline citations.

**Acceptance Scenarios**:

1. **Given** valid inputs (artifact index, test catalog, control library), **When** the user invokes the skill, **Then** the system returns `rcsa_control_narratives.md` with a summary table and per-control narratives (3–5 sentences each) with inline citations.
2. **Given** strong, sufficient evidence exists for a control, **When** the skill generates the narrative, **Then** the narrative includes 3–5 sentences with inline citations referencing real artifacts from the index.
3. **Given** multiple valid evidence sources exist for a control, **When** the skill generates the narrative, **Then** the narrative combines evidence consistently and cites all relevant artifacts.

---

### User Story 2 — Detect and Flag Compliance Gaps (Priority: P1)

A risk/compliance team member wants the system to strictly enforce evidence requirements so that missing or insufficient controls are clearly flagged as risks. If no sufficient evidence exists for a control, the system MUST output a Gap statement and MUST NOT imply compliance.

**Why this priority**: The Constitution (Principle 1: Accuracy Over Speed) requires that the system never imply compliance without proof. Gap detection is fundamental to audit readiness and is required for the May 7th demo.

**Independent Test**: Can be fully tested by providing inputs where one or more controls have no supporting evidence and verifying the output contains explicit Gap flags for those controls with no compliance narrative generated.

**Acceptance Scenarios**:

1. **Given** valid inputs where no evidence exists for a control, **When** the skill processes the inputs, **Then** the output includes "Gap: No evidence found for [control]."
2. **Given** valid inputs where evidence is mapped but does not match the expected evidence types defined in the control library, **When** the skill generates output, **Then** the system outputs a Gap and does NOT generate a compliance narrative for that control.
3. **Given** valid inputs where all controls lack sufficient evidence, **When** the skill processes the inputs, **Then** all four controls are flagged as Gap with clear explanations.

---

### User Story 3 — Validate Citations and Ensure Traceability (Priority: P1)

A risk/compliance team member wants all citations validated against real artifacts so that the documentation can be trusted for audit use. Every citation in the narratives must resolve to a real artifact in the index. Invalid citations must be flagged and replaced.

**Why this priority**: Citation traceability is a core audit requirement. Without it, the generated documentation has no evidentiary value. Required for the May 7th demo.

**Independent Test**: Can be fully tested by generating narratives and running citation validation, then verifying the `validation_report.md` accurately reports total, valid, invalid, and coverage percentage.

**Acceptance Scenarios**:

1. **Given** generated narratives with citations, **When** the system runs validation, **Then** all citations resolve to real artifacts in the index.
2. **Given** invalid citations exist in the output, **When** the system validates the output, **Then** invalid citations are removed and replaced with Gap statements.
3. **Given** validation completes, **When** the system finishes execution, **Then** `validation_report.md` is generated with total citations, valid citations, invalid citations, and coverage %.

---

### User Story 4 — Graceful Degradation When LLM Fails (Priority: P1)

If the LLM is unavailable or returns malformed output, the system must still produce structured outputs. All controls are marked as Gap with placeholder text, and the validation report explains the failure.

**Why this priority**: The Constitution (Principle 2: Graceful Degradation) requires that scripts still function without the LLM. The system loses narratives but keeps the data — which is better than total failure in an audit context. Required for the May 7th demo.

**Independent Test**: Can be fully tested by simulating an LLM failure (empty or malformed response) and verifying the system still produces both output files with all controls marked as Gap and the validation report explaining the failure.

**Acceptance Scenarios**:

1. **Given** the LLM is unavailable or returns an error, **When** the system detects the failure, **Then** the system still produces `rcsa_control_narratives.md` and `validation_report.md`.
2. **Given** the LLM output is missing or malformed, **When** the system validates the output, **Then** all controls are marked as Gap with placeholder text explaining the failure.
3. **Given** degraded output is produced, **When** the user reviews the output, **Then** the validation report clearly explains that LLM processing failed and outputs are placeholders.

---

### Edge Cases

- **Missing artifact index, test catalog, or control library**: System returns an error specifying which required input is missing; pipeline stops. User can correct without restarting.
- **No artifacts found in any input file**: All four controls are flagged as Gap.
- **Evidence mapped but does not match expected evidence types in control library**: System treats as insufficient and outputs Gap. (Definition: evidence is insufficient when mapped artifacts don't match the expected evidence types defined in the control library for that control.)
- **All citations in output are invalid**: Validation report shows 0% coverage; output is flagged as unreliable. Report is not suppressed.
- **Duplicate or conflicting artifacts in inputs**: System uses available evidence conservatively and may output Gap.
- **LLM returns fewer than 4 controls**: System detects missing controls and fills with Gap placeholders.
- **LLM returns narratives without citations**: System flags as invalid; replaced with Gap statements.
- **LLM rate limit exceeded or timeout**: System applies graceful degradation — produces structured output with Gap placeholders and explains failure in validation report.
- **LLM returns malformed JSON response**: System validates response format before parsing; invalid responses trigger graceful degradation.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-002**: System MUST accept artifact indexes (JSON) listing all discovered files, functions, and classes in a repository
- **FR-003**: System MUST accept test catalogs (JSON) listing test names, file paths, and pass/fail status
- **FR-004**: System MUST accept a control library (JSON) defining controls, objectives, and expected evidence types
- **FR-005**: All input validation — file format, schema structure — MUST be handled by deterministic script logic before any LLM processing begins
- **FR-015**: System MUST build an index of all repository artifacts (code files, tests, configs)
- **FR-016**: System MUST map discovered evidence to defined controls (Access Control, Change Management, Data Quality, Incident Handling)
- **FR-017**: System MUST generate auditor-friendly narratives (3–5 sentences per control) with inline citations
- **FR-018**: System MUST explicitly flag a "Gap" statement when no sufficient evidence exists for a control — system MUST NOT imply compliance without proof
- **FR-019**: System MUST validate that every citation resolves to a real artifact in the index
- **FR-020**: System MUST produce a summary table showing which controls have evidence vs. gaps
- **FR-021**: System MUST produce `rcsa_control_narratives.md` as the primary output; structure and formatting enforced by a defined output template
- **FR-022**: System MUST produce `validation_report.md` showing citation resolution stats (total, valid, invalid, coverage %)

### Key Entities

- **Artifact Index**: JSON listing repository files, functions, and classes. Primary input for evidence discovery.
- **Test Catalog**: JSON listing test names, file paths, and pass/fail status. Input for evidence mapping.
- **Control Library**: JSON defining the four compliance controls (AC, CM, DQ, IH), their objectives, and expected evidence types. Input for mapping rules.
- **Artifact Registry**: Normalized, structured registry of all artifacts and tests built from inputs. Intermediate data used as source of truth for citation validation.
- **Mapped Evidence**: Structured evidence mapped to each of the four controls. Intermediate data passed to the LLM for narrative generation.
- **Control Narrative**: 3–5 sentence auditor-friendly paragraph per control with inline citations. Primary output component.
- **Gap Statement**: Explicit flag when evidence is missing or insufficient for a control. Output component.
- **Validation Report**: Report showing citation resolution statistics — total, valid, invalid, coverage %. Secondary output.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-005**: ≥ 85% of true compliance gaps (controls with missing/weak evidence) are correctly flagged in the RCSA control narratives
- **SC-006**: ≤ 5% of controls or tests referenced in RCSA narratives are invented (not present in the source artifacts)
- **SC-007**: ≥ 90% of citations in RCSA control narratives resolve to real artifacts in the repository index
- **SC-008**: ≥ 3 out of 4 compliance controls are addressed (with evidence or explicit Gap flag) in every RCSA run. Note: the system MUST attempt all 4 controls every run; this metric measures quality tolerance across varied inputs.
- **SC-009**: ≥ 90% structural match when the same skill is run on the same inputs multiple times

## Assumptions

- Inputs are valid JSON files. Mitigation: input validation scripts enforce structure before LLM processing (FR-005).
- The control library defines expected evidence types for each control. Mitigation: mapping logic uses the control library as the source of truth.
- The LLM may produce errors, hallucinations, or malformed output. Mitigation: validation scripts replace invalid outputs with Gap statements (FR-019).
- All four controls must be evaluated every run. Mitigation: system enforces all four controls; missing controls are filled with Gap placeholders.
- Evidence sufficiency is determined by matching against the control library's expected evidence types. Mitigation: when evidence doesn't match expected types, the system flags a Gap.
- Demo uses small, representative datasets. Mitigation: system is tested with sample inputs provided in assets.
- The system does not store inputs permanently. Outputs are stored locally as generated Markdown files (Constitution Section 3).
