# F5 — SKILL.md: RCSA Control Narrative Generation

## Feature Specification

---

## 1. Feature Identity

| Field             | Value                                                                                                                                                                   |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Feature Name      | F5 — SKILL.md: RCSA Control Narrative Generation                                                                                                                        |
| Feature Branch    | f5-skill.md-rcsa-control-narratives                                                                                                                                     |
| Created           | 2026-04-02                                                                                                                                                              |
| Status            | Draft                                                                                                                                                                   |
| Owner             | Whitney Robinson (PM), Sheila Green, Molly Lowell                                                                                                                       |
| Demo Day Deadline | May 7, 2026                                                                                                                                                             |
| Input             | User provides artifact index (JSON), test catalog (JSON), and control library (JSON). Scripts parse these inputs and pass structured evidence to the LLM for reasoning. |

---

## 2. User Scenarios

**Note:** All LLM-generated output is a draft intended for human review. Narratives must include citations or explicit Gap statements. The system follows a strict rule: if evidence is missing or insufficient, the system MUST output a Gap and MUST NOT imply compliance.

---

### US-1: Generate Evidence-Backed Control Narratives (P1)

As a risk/compliance team member, I want the system to generate control narratives for four controls — Access Control (AC), Change Management (CM), Data Quality (DQ), and Incident Handling (IH) — using repository evidence, so that I can assess compliance without manually reviewing the codebase.

**Priority:** P1 — Required for demo day.

**What this tests:**
The system’s ability to map evidence (code files, tests, configurations) to controls and generate clear, citation-backed narratives.

#### Acceptance Scenarios:

| #   | Given                                                   | When                               | Then                                                                                  |
| --- | ------------------------------------------------------- | ---------------------------------- | ------------------------------------------------------------------------------------- |
| 1.1 | Valid artifact index, test catalog, and control library | The system processes the inputs    | A narrative is generated for each of the four controls (AC, CM, DQ, IH)               |
| 1.2 | Strong, sufficient evidence exists for a control        | The system generates the narrative | The narrative includes 3–5 sentences with inline citations referencing real artifacts |
| 1.3 | Multiple valid evidence sources exist                   | The system generates the narrative | The narrative combines evidence consistently and cites all relevant artifacts         |

---

### US-2: Detect and Flag Compliance Gaps (P1)

As a risk/compliance team member, I want the system to strictly enforce evidence requirements, so that missing or insufficient controls are clearly flagged as risks.

**Priority:** P1 — Required for demo day.

**What this tests:**
The system’s ability to enforce strict audit rules: no sufficient evidence = Gap.

#### Acceptance Scenarios:

| #   | Given                                     | When                            | Then                                                                  |
| --- | ----------------------------------------- | ------------------------------- | --------------------------------------------------------------------- |
| 2.1 | No evidence exists for a control          | The system processes the inputs | Output includes “Gap: No evidence found for [control]”                |
| 2.2 | Evidence is weak, indirect, or incomplete | The system generates output     | The system outputs a Gap and does NOT generate a compliance narrative |
| 2.3 | All controls lack sufficient evidence     | The system processes inputs     | All controls are flagged as Gap with clear explanations               |

---

### US-3: Validate Citations and Ensure Traceability (P1)

As a risk/compliance team member, I want all citations validated against real artifacts, so that I can trust the documentation for audit use.

**Priority:** P1 — Required for demo day.

**What this tests:**
The system’s ability to validate citations and enforce traceability.

#### Acceptance Scenarios:

| #   | Given                               | When                          | Then                                                                         |
| --- | ----------------------------------- | ----------------------------- | ---------------------------------------------------------------------------- |
| 3.1 | Generated narratives with citations | The system runs validation    | All citations resolve to real artifacts in the index                         |
| 3.2 | Invalid citations exist             | The system validates output   | Invalid citations are removed and replaced with Gap statements               |
| 3.3 | Validation completes                | The system finishes execution | validation_report.md is generated with total, valid, invalid, and coverage % |

---

## 3. Edge Cases

| Edge Case                                                | Expected Behavior                                                       |
| -------------------------------------------------------- | ----------------------------------------------------------------------- |
| Missing artifact index, test catalog, or control library | System returns error specifying missing required input                  |
| No artifacts found                                       | All controls are flagged as Gap                                         |
| Weak or partial evidence                                 | System treats as insufficient and outputs Gap                           |
| All citations invalid                                    | Validation report shows 0% coverage and output is flagged as unreliable |
| Duplicate or conflicting artifacts                       | System uses available evidence conservatively and may output Gap        |

---

## 4. Requirements

### 4.1 Functional Requirements

* **FR-002**: System MUST accept artifact indexes (JSON) listing all discovered files, functions, and classes in a repository
* **FR-003**: System MUST accept test catalogs (JSON) listing test names, file paths, and pass/fail status
* **FR-004**: System MUST accept a control library (JSON) defining controls, objectives, and expected evidence types
* **FR-005**: All input validation MUST be handled by deterministic scripts before LLM processing
* **FR-015**: System MUST build an index of all repository artifacts (code, tests, configs)
* **FR-016**: System MUST map discovered evidence to defined controls (AC, CM, DQ, IH)
* **FR-017**: System MUST generate auditor-friendly narratives (3–5 sentences per control) with inline citations
* **FR-018**: System MUST explicitly flag a “Gap” when no sufficient evidence exists — MUST NOT imply compliance
* **FR-019**: System MUST validate that every citation resolves to a real artifact in the index
* **FR-020**: System MUST produce a summary table showing which controls have evidence vs gaps
* **FR-021**: System MUST produce rcsa_control_narratives.md as the primary output
* **FR-022**: System MUST produce validation_report.md with citation resolution stats

---

### 4.2 Key Entities

* **Artifact Index**: JSON listing repository files, functions, and classes (input)
* **Test Catalog**: JSON listing test names, file paths, and pass/fail status (input)
* **Control Library**: JSON defining compliance controls and expected evidence types (input)
* **Control Narrative**: 3–5 sentence explanation per control with citations (output)
* **Validation Report**: Report showing citation resolution statistics (output)

---

## 5. Pipeline Flow

```text
Inputs (artifact index, test catalog, control library)
        ↓
Scripts parse and validate inputs
        ↓
LLM maps evidence to controls and generates narratives
        ↓
Validation script checks all citations
        ↓
Final outputs:
- rcsa_control_narratives.md
- validation_report.md
```

---

## 6. Boundaries — What F5 Does NOT Do

| Responsibility              | Owned By               |
| --------------------------- | ---------------------- |
| Parsing JSON inputs         | Scripts                |
| Building artifact index     | Scripts                |
| Citation validation logic   | Scripts                |
| Output file formatting      | Templates (Assets)     |
| Data storage or persistence | System (outside skill) |

---

## 7. Constitution Constraints Active on F5

* Accuracy Over Speed — all claims must be evidence-backed
* Audit-Ready by Default — every output must include citations or gaps
* Never imply compliance without proof
* Simplicity First — file-based, no UI
* Graceful Degradation — scripts still function without LLM

---

## 8. Processing Approach

* All inputs are processed together to allow full context
* LLM receives structured artifact + test data
* LLM applies mapping logic to controls
* Scripts validate outputs before final generation

---

## 9. Success Criteria

| SC ID  | What We Measure                                 | Target         |
| ------ | ----------------------------------------------- | -------------- |
| SC-005 | ≥ 85% of true compliance gaps correctly flagged | Risk Detection |
| SC-006 | ≤ 5% of referenced artifacts are invented       | Output Quality |
| SC-007 | ≥ 90% of citations resolve to real artifacts    | Traceability   |
| SC-008 | ≥ 3 of 4 controls addressed per run             | Output Quality |
| SC-009 | ≥ 90% consistency across repeated runs          | Reliability    |

---

## 10. Approved Assumptions

| # | Assumption                           | Mitigation                                   |
| - | ------------------------------------ | -------------------------------------------- |
| 1 | Inputs are valid JSON                | Input validation scripts enforce structure   |
| 2 | Evidence mapping logic is consistent | Defined control library and strict rules     |
| 3 | LLM may produce errors               | Validation replaces invalid outputs with Gap |
| 4 | All controls must be evaluated       | System enforces all four controls every run  |

---

## 11. Open Items

| # | Item                                    | Status    |
| - | --------------------------------------- | --------- |
| 1 | Exact citation format standardization   | ⏳ Pending |
| 2 | Validation threshold tuning             | ⏳ TBD     |
| 3 | Narrative tone alignment with Synchrony | ⏳ TBD     |

---
