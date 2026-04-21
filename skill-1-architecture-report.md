# Architecture Review Report

**Generated:** 2026-04-20 20:31:01  
**Artifacts reviewed:** 6  
**Total issues found:** 6  
**Critical:** 1  |  **High:** 2  |  **Medium:** 2  |  **Low:** 1

---

## 0. Artifact Discovery and Summary

### Discovery status

- **Architecture docs found:** `.cursor/skills/ARCHITECTURE.md`, `.cursor/skills/scout/examples/ARCHITECTURE.md`, `starter-repo/.github/skills/scout/examples/ARCHITECTURE.md`
- **Constitution docs found:** `skills/data-dictionary/docs/constitution.md` (plus additional specify copies)
- **Spec docs found:** `skills/data-dictionary/docs/project-spec.md` (plus feature specs and specify copies)
- **Plan docs found:** `skills/data-dictionary/docs/data-dictionary-master.md` (plus feature plans and specify copies)
- **Missing from discovery set:** `roadmap.md`, `specs/TODO_*.md`

### Reviewed artifact summaries

- **ARCHITECTURE (`.cursor/skills/ARCHITECTURE.md`)**
  - Modules: `validate_input.py`, `extract_fields.py`, `attach_citations.py`, `add_timestamps.py`, `assemble_output.py`, `generate_qa_report.py`, `utils.py`
  - Dependency shape: stage scripts depend on `utils.py` (hub-and-spoke)
  - Data flow: validate -> extract -> attach/merge -> timestamp -> render outputs
  - Tech stack: Python stdlib only
  - Hotspots: `utils.py` high fan-in

- **Constitution (`skills/data-dictionary/docs/constitution.md`)**
  - Principles: accuracy over speed, graceful degradation, simplicity-first file-based scripts, audit-ready outputs
  - Constraints: no microservices/frontend, Python-first, local file outputs
  - Security boundaries: never send PII/secrets/full source/raw datasets; outputs must remain traceable

- **Spec (`skills/data-dictionary/docs/project-spec.md`)**
  - Required Data Dictionary capability: end-to-end generation from JSON schema extracts
  - Required output fields include field metadata + description + evidence_refs + confidence + `last_verified`
  - Output artifacts: `data_dictionary.md` and `qa_report.md`
  - Acceptance constraints: citations for every field, clarification flags for ambiguous fields, deterministic validation before LLM

- **Plan (`skills/data-dictionary/docs/data-dictionary-master.md`)**
  - Feature split: F1 (LLM reasoning), F2 (deterministic scripts), F3 (assets/templates)
  - Explicit pipeline contracts between extracted fields, LLM 5-key output, merged records, timestamped records
  - Graceful degradation + retry behavior owned by scripts (F2), not by prompt-only behavior

- **Skill under review (`.cursor/skills/data_dictionary/SKILL.md`)**
  - Contains both pipeline orchestration steps and F1 LLM reasoning contract
  - Enforces strict 5-key JSON output from LLM
  - Includes scoring rules, edge-case handling, and guardrails

---

## 1. Design Principle Violations

| # | Severity | Principle | Violation | Recommendation |
|---|----------|-----------|-----------|----------------|
| 1 | High | Simplicity First + clear feature ownership (constitution + master plan) | `SKILL.md` mixes F1 reasoning guidance with full F2 orchestration and file-system execution instructions, while also claiming in Scope that it does not run downstream steps. This creates contradictory ownership boundaries for runtime behavior. | Split the document into two explicit sections: (a) orchestration contract (if intentionally owned here), and (b) F1 reasoning contract; or remove orchestration from this skill and keep only F1 instructions consistent with plan ownership. |
| 2 | Medium | Accuracy Over Speed / Audit-ready by default | The skill does not explicitly encode degraded-mode signaling (LLM unavailable) despite constitution and plan emphasizing graceful degradation with transparent reporting. | Add explicit failure-state behavior in `SKILL.md` that aligns with F2 fallback output and QA reporting expectations, including when to return placeholders and how to label run status. |

---

## 2. Spec Coverage Gaps

| # | Severity | Spec | Required Capability | Architecture Status | Recommendation |
|---|----------|------|---------------------|---------------------|----------------|
| 1 | Critical | `project-spec.md` + `data-dictionary-master.md` repository contract | Runtime paths and asset locations must align with canonical repo layout (`skills/data-dictionary/...`, root `assets/...`). | `SKILL.md` hardcodes `.cursor/skills/data_dictionary/...` absolute paths and local assets that conflict with documented canonical layout. This can prevent the documented build from executing in intended repo structure. | Replace hardcoded absolute `.cursor/...` paths with contract-aligned relative paths or centralized path constants; align asset paths with documented F3 location. |
| 2 | Medium | FR-008 + Story 1 acceptance criteria | Every field description must carry citation to source schema artifact. | `SKILL.md` enforces non-empty `evidence_refs` but does not require `source_path`-anchored citation in each field-level evidence set. | Require at least one evidence entry explicitly tied to `source_path` (or equivalent source artifact reference) per field. |

### Spec Coverage Summary

| Spec File | Features Checked | Covered | Gaps | Coverage % |
|-----------|------------------|---------|------|------------|
| `skills/data-dictionary/docs/project-spec.md` | Data dictionary F1/F2/F3 contracts, output requirements, citations, paths | 4 | 2 | 67% |

---

## 3. Plan Alignment Issues

| # | Severity | Plan Item | Architecture Status | Recommendation |
|---|----------|-----------|---------------------|----------------|
| 1 | High | `data-dictionary-master.md` says F2 owns deterministic retries/degradation and pipeline execution details | `SKILL.md` currently defines end-to-end script execution and storage paths while also declaring it does not run downstream steps; this is internally contradictory and plan-misaligned. | Make plan ownership explicit in skill text and remove contradictory Scope statements; preserve only LLM contract here if following strict F1 ownership. |
| 2 | Low | Plan states templates are authoritative and location-specific | `SKILL.md` lists template assets in a skill-local directory that differs from plan's root `assets/` placement. | Cross-reference the master plan asset directory in `SKILL.md` and avoid duplicate, divergent asset location documentation. |

### Plan Coverage Summary

| Architecture Component | Planned? | Status |
|------------------------|----------|--------|
| LLM 5-key contract | Yes (F1) | Aligned |
| Script sequence | Yes (F2) | Partially aligned (ownership/path mismatch) |
| Graceful degradation semantics | Yes (F2) | Partially aligned (implicit in skill, not explicit contract) |
| Asset path contract | Yes (F3) | Misaligned |

---

## 4. Internal Architecture Issues

| # | Severity | Issue | Location | Recommendation |
|---|----------|-------|----------|----------------|
| 1 | Medium | Contradictory statements inside `SKILL.md`: pipeline section instructs running validation/extraction/assembly scripts, but Scope says the skill does not parse schema files, does not assemble outputs, and does not run downstream steps. | `.cursor/skills/data_dictionary/SKILL.md` (`Pipeline Orchestration` vs `Scope`) | Keep one authoritative behavior model; either this skill orchestrates full pipeline, or it is strictly the F1 reasoning layer. |

---

## 5. Suggested Fixes

### ARCHITECTURE.md updates

- Add a short "Contract Alignment" subsection clarifying whether this architecture describes the canonical runtime under `skills/data-dictionary/` or an IDE-local `.cursor` variant.
- Document a single authoritative path strategy (absolute vs relative) and reference the master plan path contract.

### Spec updates

- In `project-spec.md`, add one line clarifying whether local `.cursor` skill packaging is allowed as an implementation variant, and which path contract is normative for acceptance.
- In FR-008 language, explicitly state whether citation must include `source_path` per field or if any evidence string is acceptable.

### Plan updates

- In `data-dictionary-master.md`, add a short compatibility note describing how SKILL.md variants (repo canonical vs IDE-local wrapper) should remain synchronized.
- Add a review gate check that rejects contradictory Scope/orchestration statements in F1 documents.
