# Feature Specification: F5 — RCSA SKILL.md (LLM Agent Instructions)

**Feature Branch**: `f5-rcsa-skill`  
**Created**: 2026-04-07  
**Status**: Draft  
**Input**: User description: "Create the SKILL.md agent instruction file that guides the LLM to generate citation-backed RCSA control narratives from pre-processed mapped evidence, detect compliance gaps, and produce audit-ready documentation following defined output templates."

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Generate Citation-Backed Control Narratives (Priority: P1)

A risk/compliance team member has pre-processed mapped evidence (provided by the deterministic scripts pipeline) for four compliance controls: Access Control, Change Management, Data Quality, and Incident Handling. The LLM reads the SKILL.md instructions and generates 3–5 sentence auditor-friendly narratives per control, with inline citations referencing real artifacts from the repository.

**Why this priority**: This is the core reasoning task that only the LLM can perform. Without narrative generation, the skill produces raw data but no audit-ready documentation. Required for the May 7th demo.

**Independent Test**: Can be fully tested by providing sample mapped evidence for all four controls and verifying the LLM produces a narrative for each control with inline citations that reference real artifacts.

**Acceptance Scenarios**:

1. **Given** mapped evidence for all four controls is available, **When** the LLM processes the evidence using SKILL.md instructions, **Then** the output contains a 3–5 sentence narrative per control with inline citations.
2. **Given** mapped evidence with multiple artifacts per control, **When** the LLM generates narratives, **Then** the narratives combine evidence from multiple sources and cite all relevant artifacts.
3. **Given** mapped evidence in the expected structured format, **When** the LLM processes it, **Then** the output follows the narrative template structure defined in assets (F4).

---

### User Story 2 - Detect and Flag Compliance Gaps (Priority: P1)

When mapped evidence is missing or insufficient for a control, the LLM must output an explicit Gap statement. The LLM must never imply compliance without proof. This is a hard rule from the project Constitution (Principle 1: Accuracy Over Speed).

**Why this priority**: Gap detection is fundamental to audit readiness. A compliance tool that implies compliance without evidence is worse than no tool at all. Required for the May 7th demo.

**Independent Test**: Can be fully tested by providing mapped evidence where one or more controls have no supporting artifacts and verifying the LLM outputs explicit Gap flags for those controls with no compliance narrative.

**Acceptance Scenarios**:

1. **Given** mapped evidence where no artifacts exist for a control, **When** the LLM processes the evidence, **Then** the output includes "Gap: No evidence found for [control]" and does NOT generate a compliance narrative for that control.
2. **Given** mapped evidence where artifacts are mapped but do not match expected evidence types, **When** the LLM processes the evidence, **Then** the output flags a Gap for that control.
3. **Given** mapped evidence where all four controls lack sufficient evidence, **When** the LLM processes the evidence, **Then** all four controls are flagged as Gap with clear explanations.

---

### User Story 3 - Produce Summary Table (Priority: P1)

The LLM produces a summary table at the top of the output showing which controls have evidence-backed narratives and which have Gap flags, giving the user an at-a-glance compliance overview before the detailed narratives.

**Why this priority**: The summary table is the first thing an auditor or compliance officer sees. It provides immediate visibility into compliance posture without reading full narratives. Required for the May 7th demo.

**Independent Test**: Can be fully tested by running the skill with sample inputs and verifying the output begins with a summary table listing all four controls with their status (Evidence or Gap).

**Acceptance Scenarios**:

1. **Given** the LLM has generated narratives and/or Gap flags for all four controls, **When** the output is assembled, **Then** a summary table appears at the top listing each control and its status.
2. **Given** a mix of evidence-backed and gap-flagged controls, **When** the summary table is generated, **Then** it accurately reflects which controls have evidence and which have gaps.

---

### User Story 4 - Follow Output Templates (Priority: P2)

The LLM structures its output according to the templates defined in assets (F4), ensuring consistent formatting across runs. The narrative template defines the structure of `rcsa_control_narratives.md` and the validation report template defines the structure of `validation_report.md`.

**Why this priority**: Consistent output formatting is important for professional audit documentation, but the skill still delivers value even with minor formatting variations. Prioritized after core narrative and gap detection functionality.

**Independent Test**: Can be fully tested by comparing LLM output against the defined templates and verifying structural match (headings, sections, table format).

**Acceptance Scenarios**:

1. **Given** output templates are available in assets, **When** the LLM generates output, **Then** the output structure matches the template (correct headings, sections, table format).
2. **Given** the LLM generates output for a run with mixed evidence and gaps, **When** the output is compared to the template, **Then** all required sections are present and correctly ordered.

---

### Edge Cases

- What happens when the mapped evidence format is unexpected or malformed? The SKILL.md must instruct the LLM to flag the issue rather than guess at the data.
- What happens when evidence is ambiguous and could apply to multiple controls? The SKILL.md must instruct the LLM to cite conservatively and note the ambiguity.
- What happens when the LLM is unsure about evidence sufficiency? The SKILL.md must instruct the LLM to flag a Gap rather than imply compliance.
- What happens when the LLM receives an empty evidence set for all controls? The SKILL.md must instruct the LLM to flag all four controls as Gap.
- What happens when the LLM generates narratives without citations? Post-processing validation (F6 — Scripts) catches this, but the SKILL.md should explicitly instruct the LLM to always include citations.
- How does the system handle the LLM being unavailable or returning errors? Graceful degradation is handled by the scripts (F6), not the SKILL.md itself.

---

## Requirements *(mandatory)*

### Functional Requirements

| ID | Requirement |
|----|-------------|
| **FR-001** | SKILL.md MUST instruct the LLM to generate auditor-friendly narratives (3–5 sentences per control) with inline citations referencing real artifacts |
| **FR-002** | SKILL.md MUST instruct the LLM to explicitly flag a "Gap" statement when no sufficient evidence exists for a control |
| **FR-003** | SKILL.md MUST instruct the LLM to never imply compliance without proof |
| **FR-004** | SKILL.md MUST instruct the LLM to produce a summary table showing which controls have evidence vs. gaps |
| **FR-005** | SKILL.md MUST instruct the LLM to process all four controls (Access Control, Change Management, Data Quality, Incident Handling) in every run |
| **FR-006** | SKILL.md MUST instruct the LLM to use only evidence provided in the mapped evidence input — no external knowledge or assumptions about the repository |
| **FR-007** | SKILL.md MUST instruct the LLM to cite artifacts using the exact identifiers from the artifact registry |
| **FR-008** | SKILL.md MUST instruct the LLM to prefer Gap flags over uncertain compliance claims |
| **FR-009** | SKILL.md MUST instruct the LLM to follow the output templates defined in assets (F4) for consistent formatting |
| **FR-010** | SKILL.md MUST define the sequence of tool calls the agent should make (validate inputs → build registry → map evidence → generate narratives → validate citations) |

### Key Entities

| Entity | Description |
|--------|-------------|
| **SKILL.md** | The natural language instruction file that the LLM agent reads to understand its workflow, rules, and output expectations. This is the primary deliverable of F5. |
| **Mapped Evidence** | Structured evidence mapped to each of the four controls, provided as input to the LLM by the scripts pipeline (F6). Contains artifact identifiers, file paths, evidence types, and control associations. |
| **Control Narrative** | 3–5 sentence auditor-friendly paragraph per control with inline citations. Primary output of the LLM reasoning step. |
| **Gap Statement** | Explicit flag when evidence is missing or insufficient for a control. Output when the LLM cannot support a compliance claim. |
| **Summary Table** | At-a-glance table at the top of the output showing all four controls and their status (Evidence or Gap). |
| **Output Templates** | Markdown templates (owned by F4 — Assets) that define the structure and formatting of the final output files. |

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

| ID | Criteria |
|----|----------|
| **SC-001** | ≥ 85% of true compliance gaps (controls with missing or weak evidence) are correctly flagged in the output |
| **SC-002** | ≤ 5% of artifacts referenced in narratives are invented (not present in the mapped evidence input) |
| **SC-003** | All 4 compliance controls are addressed (with evidence narrative or explicit Gap flag) in every run |
| **SC-004** | ≥ 90% structural match when the same SKILL.md is run on the same inputs multiple times |
| **SC-005** | Output follows the defined template structure with all required sections present |

---

## Assumptions

- The LLM receives pre-processed, structured mapped evidence — not raw repository files. Input preparation is handled by F6 (Scripts).
- The deterministic pipeline (F6 — Scripts) handles all input validation and evidence mapping before the LLM is invoked.
- The LLM does not have direct access to the repository — it works only with the evidence provided to it.
- The SKILL.md defines the expected output structure (summary table, per-control narratives, gap flags). F4 (Assets) will later formalize these into reusable templates based on what F5 establishes. During F5 development, output structure is defined inline within the SKILL.md instructions. 
- The LLM may hallucinate citations; post-processing validation (F6 — Scripts) catches and corrects these after the LLM generates output.
- Graceful degradation when the LLM fails is handled by F6 (Scripts), not by the SKILL.md itself.
- The SKILL.md will be used within the Claude Agent Skills runtime environment.

