# Implementation Plan: F4 — RCSA Assets (Templates, Sample Data, and Control Library)

**Branch**: `f4-rcsa-assets` | **Date**: 2026-04-07 | **Spec**: [spec.md](./spec.md)  
**Input**: Feature specification from `/specs/f4-rcsa-assets/spec.md`

---

## Summary

Build the assets package that formalizes the output templates, sample input data, control library, and citation format specification for the RCSA Control Narrative Generation skill. F4 is the final feature built — it takes the output structures defined inline by F5 (SKILL.md) and the pipeline requirements established by F6 (Scripts) and packages them into reusable, versioned asset files. Deliverables include two Markdown output templates (`rcsa_control_narratives_template.md`, `validation_report_template.md`), three JSON sample input files (`sample_artifact_index.json`, `sample_test_catalog.json`, `control_library.json`), and a citation format specification. All assets must be backward-compatible with F5 and F6 and pass F6's input validation with 0 errors.

---

## Technical Context

**Language/Version**: Markdown (output templates, citation format spec) and JSON (sample data, control library) — no executable code  
**Primary Dependencies**: F5 (SKILL.md) defines the output structure that templates formalize; F6 (Scripts) defines the input schemas that sample data must conform to and the citation format that the spec must match. Both are upstream dependencies — F4 formalizes what they established.  
**Storage**: Local filesystem — all assets stored as static files in `.cursor/skills/rcsa/assets/` (Constitution constraint: no database, no cloud storage)  
**Testing**: Manual validation — verify templates match F5/F6 output structure; run sample data through F6's input validation script to confirm 0 errors; verify control library schema matches F6's expected format  
**Target Platform**: Claude Agent Skills (Cursor IDE integration) — assets are consumed by the SKILL.md (F5) and scripts (F6) at runtime

**Project Type**: Static asset package (documentation and data files — no executable code)  
**Performance Goals**: N/A — assets are static files with no runtime performance requirements  
**Constraints**: Templates must not introduce new formatting or sections that F5 and F6 don't already support (backward compatibility); all JSON files must use UTF-8 encoding; sample data must exercise both narrative generation and gap detection scenarios  
**Scale/Scope**: Demo-scale — sample data represents a small repository sufficient for the May 7th demo, not enterprise-scale

---

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

**Status: PASS**

| Rule | Plan Compliance |
|------|-----------------|
| Principle 1: Accuracy Over Speed — never imply compliance without proof | ✅ Control library defines explicit expected evidence types per control; sample data includes intentional gaps to test gap detection; templates include gap flag format |
| Principle 2: Graceful Degradation — scripts must function without LLM | ✅ Not directly applicable to F4 (static assets), but sample data can be used to test F6's graceful degradation path |
| Section 3: No permanent data storage; outputs stored locally as Markdown | ✅ All assets are local static files — Markdown templates and JSON data files |
| Section 4: All input validation handled by deterministic scripts before LLM | ✅ Sample data must pass F6's input validation (FR-007); assets support the validation pipeline, they don't bypass it |
| Tech Stack: Python 3.11, Claude Sonnet | ✅ No executable code in F4; assets are consumed by Python 3.11 scripts (F6) and Claude Sonnet (F5) |
| Security: No hardcoded secrets | ✅ Assets contain no API keys, credentials, or secrets — only templates and sample data |

---

## Project Structure

### Documentation (this feature)

```text
specs/f4-rcsa-assets/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/           # Phase 1 output
└── tasks.md             # Phase 2 output (/speckit.tasks — Week 4)
```

### Source Code (repository root)

```text
.cursor/skills/rcsa/
└── assets/
    ├── rcsa_control_narratives_template.md   # FR-001: Narrative output template
    ├── validation_report_template.md         # FR-002: Validation report template
    ├── sample_artifact_index.json            # FR-003: Sample artifact index
    ├── sample_test_catalog.json              # FR-004: Sample test catalog
    ├── control_library.json                  # FR-005, FR-006: Control library (4 controls)
    └── citation_format_spec.md              # FR-009: Citation format specification
```

**Structure Decision**: Flat asset directory under `.cursor/skills/rcsa/assets/`. All asset files live in a single directory — no subdirectories needed given the small number of files (6 total). This keeps the structure simple and matches the existing skill layout where `SKILL.md` (F5) and `tools/` (F6) are siblings under `.cursor/skills/rcsa/`. No `src/` or `tests/` directories — F4 contains no executable code. Validation of assets is performed by running them through F6's existing pipeline.

---

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

*No violations. Constitution Check passed with no exceptions.*
