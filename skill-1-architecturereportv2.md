# Architecture Review Report

**Generated:** 2026-04-20 21:24:49  
**Artifacts reviewed:** 5  
**Total issues found:** 6  
**Critical:** 0  |  **High:** 3  |  **Medium:** 2  |  **Low:** 1

---

## 0. Artifact Discovery and Summary

### Discovered Artifacts

- **Review prompt:** `.cursor/prompts/review.md`
- **Skill under review:** `.cursor/skills/data_dictionary/SKILL.md`
- **Constitution:** `skills/data-dictionary/docs/constitution.md`
- **Spec:** `skills/data-dictionary/docs/project-spec.md`
- **Plan:** `skills/data-dictionary/docs/data-dictionary-master.md`
- **Architecture:** `.cursor/skills/ARCHITECTURE.md`

### Found vs Missing

- **Found:** All requested review artifacts were present and readable.
- **Missing:** No required artifact was missing for this review.

### Category Summaries

- **Architecture summary:** File-based, script-driven Python pipeline with deterministic stage scripts and shared utility layer; outputs are `data_dictionary.md` and `qa_report.md`.
- **Design principles summary:** Accuracy, audit readiness, graceful degradation, and simplicity-first with clear boundaries and minimal complexity.
- **Spec summary:** Requires deterministic validation/extraction before LLM reasoning, citations and confidence in outputs, and strict output deliverables.
- **Plan summary:** Defines clear F1/F2/F3 ownership boundaries and an 8-step end-to-end flow.

---

## 1. Design Principle Violations

| # | Severity | Principle | Violation | Recommendation |
|---|----------|-----------|-----------|----------------|
| 1 | High | Simplicity first and clear feature ownership (`constitution.md`, `data-dictionary-master.md`) | `SKILL.md` mixes F1 reasoning guidance with full F2 pipeline execution ownership, then later says it does not run downstream steps. | Refactor `SKILL.md` into two explicit blocks: (a) orchestration instructions and (b) LLM reasoning contract; remove contradictory scope lines. |
| 2 | High | Audit-ready by default (`constitution.md`) | Contradictory run instructions can lead to non-repeatable execution paths, weakening traceability expectations. | Add one authoritative execution path in `SKILL.md` and ensure all sections point to that same path. |

---

## 2. Spec Coverage Gaps

| # | Severity | Spec | Required Capability | Architecture Status | Recommendation |
|---|----------|------|---------------------|---------------------|----------------|
| 1 | Medium | `project-spec.md` FR-007 | Descriptions over 25 words should be flagged in QA, not hard-failed | `SKILL.md` currently uses strict `<=25 words` language that can be interpreted as a hard requirement. | Update `SKILL.md` wording to: "target <=25 words; over-limit descriptions are allowed and flagged by QA." |
| 2 | High | `project-spec.md` + `data-dictionary-master.md` ownership model | F1 should define reasoning/output contract; F2 scripts own deterministic processing and post-processing behaviors | `SKILL.md` combines strict F1 contract language with end-to-end script ownership and then denies downstream ownership in Scope section. | Keep F1 output contract, but make script responsibilities explicitly delegated and consistent across all sections. |
| 3 | Medium | `project-spec.md` User Story 1 flow | Input flow starts with schema upload and deterministic extraction before LLM reasoning | `SKILL.md` says expected input is pre-structured metadata, but pipeline starts with raw schema upload and extraction. | Clarify in `SKILL.md` that it supports schema-driven invocation and consumes extracted metadata after extraction stage. |

### Spec Coverage Summary

| Spec File | Features Checked | Covered | Gaps | Coverage % |
|-----------|------------------|---------|------|------------|
| `project-spec.md` | F1/F2/F3 ownership, FR-007/008/012/013, data-dictionary flow | 4 | 3 | 57% |

---

## 3. Plan Alignment Issues

| # | Severity | Plan Item | Architecture Status | Recommendation |
|---|----------|-----------|---------------------|----------------|
| 1 | Low | `data-dictionary-master.md` naming and ownership consistency | Naming and responsibility language is inconsistent inside `SKILL.md` (`data-dictionary` vs `data_dictionary`, plus mixed ownership phrasing). | Normalize naming and add a one-line ownership note at top of `SKILL.md` (what F1 owns vs what scripts own). |

### Plan Coverage Summary

| Architecture Component | Planned? | Status |
|------------------------|----------|--------|
| `validate_input.py` | Yes | Aligned |
| `extract_fields.py` | Yes | Aligned |
| `attach_citations.py` | Yes | Aligned |
| `add_timestamps.py` | Yes | Aligned |
| `assemble_output.py` | Yes | Aligned |
| `generate_qa_report.py` | Yes | Aligned |
| Skill boundary contract clarity | Yes | Misaligned in `SKILL.md` wording |

---

## 4. Internal Architecture Issues

| # | Severity | Issue | Location | Recommendation |
|---|----------|-------|----------|----------------|
| 1 | High | Direct internal contradiction: "run full pipeline" vs "does not run downstream merge/timestamp/template steps." | `.cursor/skills/data_dictionary/SKILL.md` | Remove contradiction by rewriting the Scope section to match the actual pipeline steps. |

---

## 5. Suggested Fixes

### `SKILL.md` concrete fixes (skill-only)

1. **Resolve scope contradiction**
   - In `Scope`, remove lines that say the skill "does not run downstream merge/timestamp/template steps" if pipeline orchestration remains in this skill.
   - Keep one consistent ownership statement across `Pipeline Orchestration`, `When to use`, and `Scope`.

2. **Make execution model explicit**
   - Add a `Runtime Mode` subsection with one source-of-truth flow:
     - schema input -> validate -> extract -> LLM reasoning -> attach citations -> add timestamps -> assemble output -> generate QA.
   - Remove any alternate wording that implies metadata-only invocation unless intentionally supported as a second mode.

3. **Align FR-007 wording in output contract**
   - Change strict `description <=25 words` phrasing to target-based language:
     - "Aim for <=25 words; if longer, keep evidence-grounded wording and allow QA report to flag."

4. **Clarify input contract sequencing**
   - Keep the per-field reasoning contract for Step 4, but preface it with:
     - "This contract applies after `extract_fields.py` emits `extracted_fields.json`."

5. **Add deterministic self-check gate**
   - Keep Step 7 checks and add one final line:
     - "If any check fails, emit valid 5-key fallback objects per field rather than partial output."

6. **Standardize naming inside skill docs**
   - Pick one canonical style for references in `SKILL.md` (`data_dictionary` or `data-dictionary`) and use it consistently throughout the file.
