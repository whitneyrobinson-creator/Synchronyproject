# Implementation Plan: RCSA Control Narrative Generation

**Branch**: `f5-skill-rcsa-control-narratives` | **Date**: 2026-04-02 | **Spec**: `../spec.md`
**Input**: Feature specification from `.specify/specs/f5-skill-rcsa-control-narratives/`

**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/plan-template.md` for the execution workflow.

---

## Summary

Build an RCSA skill that takes structured JSON inputs — artifact index, test catalog, and control library — and generates audit-ready control narratives for four controls: Access Control, Change Management, Data Quality, and Incident Handling. The system maps evidence to controls, generates citation-backed narratives, explicitly flags gaps where evidence is missing, and produces a validation report showing citation resolution statistics.

---

## Technical Context

**Language/Version**: Python 3.11
**Primary Dependencies**: Python standard library, JSON processing, Claude Agent Skills / SKILL.md workflow
**Storage**: Local files (JSON inputs and Markdown outputs)
**Testing**: pytest
**Target Platform**: Local CLI / repository-based execution
**Project Type**: CLI / file-based skill
**Performance Goals**: Generate outputs (`rcsa_control_narratives.md` and `validation_report.md`) within seconds for demo-sized inputs
**Constraints**: Local-only processing; no external APIs; must not imply compliance without proof; all citations must resolve to real artifacts; no sensitive data sent to LLM
**Scale/Scope**: Single-user demo workflow evaluating four controls using structured JSON inputs

---

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

**Status: PASS**

| Rule                                 | Plan Compliance                                           |
| ------------------------------------ | --------------------------------------------------------- |
| Accuracy Over Speed                  | ✅ Prioritizes validation and correctness over speed       |
| Graceful Degradation                 | ✅ System still validates and flags gaps even if LLM fails |
| Simplicity First                     | ✅ File-based, no unnecessary complexity                   |
| Audit-Ready by Default               | ✅ Outputs are structured and audit-friendly               |
| Never imply compliance without proof | ✅ Explicit gap logic enforced                             |
| No sensitive data sent to LLM        | ✅ Only structured metadata is used                        |

---

## Project Structure

### Documentation (this feature)

```text
specs/f5-skill-rcsa-control-narratives/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
└── tasks.md
```

---

### Source Code (repository root)

```text
src/
├── models/
├── services/
├── cli/
└── lib/

tests/
├── contract/
├── integration/
└── unit/
```

---

**Structure Decision**: Single-project structure using `src/` and `tests/` because this is a local, file-based skill with no frontend or distributed system requirements.

---

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

No Constitution violations identified. This section remains empty.
