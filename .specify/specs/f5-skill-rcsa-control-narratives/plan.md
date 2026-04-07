# Implementation Plan: F5 — RCSA SKILL.md (LLM Agent Instructions)

**Branch**: `f5-rcsa-skill` | **Date**: 2026-04-07 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/f5-skill-rcsa-control-narratives/spec.md`

---

## Summary

Build the SKILL.md agent instruction file that guides the LLM to generate citation-backed RCSA control narratives from pre-processed mapped evidence. The SKILL.md defines the LLM's workflow: receive structured mapped evidence for four compliance controls (Access Control, Change Management, Data Quality, Incident Handling), generate 3–5 sentence auditor-friendly narratives with inline citations, explicitly flag Gaps when evidence is missing or insufficient, and produce a summary table — all following output templates defined in F4 (Assets). The deterministic pipeline (F6 — Scripts) handles all input validation, evidence mapping, and post-processing; the SKILL.md governs only the LLM reasoning step.

---

## Technical Context

**Language/Version**: Natural language (Markdown) — SKILL.md is a prompt instruction file, not executable code  
**Primary Dependencies**: Claude Agent Skills runtime environment; output templates from F4 (Assets); mapped evidence input from F6 (Scripts)  
**Storage**: N/A — SKILL.md is a static instruction file; output files (`rcsa_control_narratives.md`, `validation_report.md`) are written by F6 (Scripts)  
**Testing**: Manual validation — run SKILL.md with sample mapped evidence inputs and verify output against acceptance scenarios in spec; automated structural validation handled by F6 (Scripts)  
**Target Platform**: Claude Agent Skills (Cursor IDE integration)

**Project Type**: Agent skill definition (prompt engineering / instruction authoring)  
**Performance Goals**: LLM generates complete output for all 4 controls in a single invocation; ≥ 90% structural consistency across repeated runs  
**Constraints**: SKILL.md must instruct the LLM to use only provided evidence (no external knowledge); must never imply compliance without proof (Constitution Principle 1); must follow output templates from F4  
**Scale/Scope**: Single SKILL.md file covering 4 compliance controls; designed for demo-scale repositories (not enterprise-scale)

---

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

**Status: PASS**

| Rule | Plan Compliance |
|------|-----------------|
| Principle 1: Accuracy Over Speed — never imply compliance without proof | ✅ FR-003 and FR-008 explicitly instruct the LLM to prefer Gap flags over uncertain claims |
| Principle 2: Graceful Degradation — scripts must function without LLM | ✅ Graceful degradation is handled by F6 (Scripts), not SKILL.md — correct separation of concerns |
| Section 3: No permanent data storage; outputs stored locally as Markdown | ✅ SKILL.md produces Markdown output; file writing handled by F6 (Scripts) to local filesystem |
| Section 4: All input validation handled by deterministic scripts before LLM | ✅ SKILL.md assumes pre-validated mapped evidence; input validation is F6 responsibility |
| Tech Stack: Python 3.11, Claude Sonnet | ✅ SKILL.md targets Claude Agent Skills runtime; no tech stack conflict |
| Security: No hardcoded secrets | ✅ SKILL.md is a static instruction file with no secrets or API keys |

---

## Project Structure

### Documentation (this feature)

```text
specs/f5-skill-rcsa-control-narratives/
├── spec.md              # Feature specification (completed)
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
├── SKILL.md                          # Primary deliverable — LLM agent instructions
└── README.md                         # Skill overview and usage documentation
```

**Structure Decision**: Single-file deliverable. F5 produces one SKILL.md file that lives in the shared `.cursor/skills/rcsa/` directory alongside assets (F4) and tools (F6). This is not a traditional source code project — it is a prompt instruction file authored in Markdown. No `src/` or `tests/` directories are needed for F5 itself. Testing is performed by running the skill with sample inputs and validating output against the spec's acceptance scenarios.

---

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

*No violations. Constitution Check passed with no exceptions.*
