# RCSA Requirements Quality Checklist

**Purpose**: Validate that the RCSA Control Narrative Generation spec is complete, clear, and ready for implementation
**Created**: 2026-04-02

---

## Requirement Completeness

* [ ] CHK001 - Are all four controls (Access Control, Change Management, Data Quality, Incident Handling) explicitly defined? [Completeness]
* [ ] CHK002 - Are all required input files (artifact index, test catalog, control library) clearly specified? [Completeness]
* [ ] CHK003 - Are output artifacts (`rcsa_control_narratives.md`, `validation_report.md`) fully described? [Completeness]
* [ ] CHK004 - Are requirements defined for handling missing or incomplete evidence? [Gap]

---

## Requirement Clarity

* [ ] CHK005 - Is the definition of a "Gap" clearly specified and unambiguous? [Clarity]
* [ ] CHK006 - Are narrative requirements (3–5 sentences per control) explicitly defined? [Clarity]
* [ ] CHK007 - Is the format of citations clearly defined? [Ambiguity]
* [ ] CHK008 - Are validation metrics (e.g., resolution stats) clearly described? [Clarity]

---

## Requirement Consistency

* [ ] CHK009 - Are citation requirements consistent across all control narratives? [Consistency]
* [ ] CHK010 - Do gap-handling rules align across all controls? [Consistency]
* [ ] CHK011 - Are validation requirements consistent with narrative generation rules? [Consistency]

---

## Acceptance Criteria Quality

* [ ] CHK012 - Are success criteria measurable (e.g., citation resolution %, gap detection)? [Measurability]
* [ ] CHK013 - Can each control narrative requirement be objectively verified? [Measurability]

---

## Scenario Coverage

* [ ] CHK014 - Are requirements defined for cases where evidence exists for all controls? [Coverage]
* [ ] CHK015 - Are requirements defined for cases where some controls lack evidence? [Coverage]
* [ ] CHK016 - Are requirements defined for cases where no evidence exists? [Gap]

---

## Edge Case Coverage

* [ ] CHK017 - Are requirements defined for invalid or malformed JSON inputs? [Edge Case, Gap]
* [ ] CHK018 - Are requirements defined for empty datasets? [Edge Case]
* [ ] CHK019 - Are requirements defined for duplicate or conflicting evidence? [Edge Case]

---

## Non-Functional Requirements

* [ ] CHK020 - Are performance expectations defined for generating outputs? [Gap]
* [ ] CHK021 - Are data privacy constraints clearly specified? [Completeness]
* [ ] CHK022 - Are reliability expectations defined for LLM outputs? [Gap]

---

## Dependencies & Assumptions

* [ ] CHK023 - Are assumptions about input data quality clearly stated? [Assumption]
* [ ] CHK024 - Are dependencies on upstream data generation (Data Dictionary skill) defined? [Dependency]

---

## Ambiguities & Conflicts

* [ ] CHK025 - Is the mapping logic between evidence and controls clearly defined? [Ambiguity]
* [ ] CHK026 - Are there any conflicting rules between narrative generation and validation? [Conflict]
