# Implementation Plan: F5 — SKILL.md (RCSA)

**Branch**: `005-skill-rcsa` | **Date**: 2026-04-10 | **Spec**: `/specs/005-skill-rcsa/spec.md`

**Input**: Feature specification from `/specs/005-skill-rcsa/spec.md`

---

## Summary

F5 (SKILL.md — RCSA) defines the natural language instructions that guide an LLM agent through a Risk Control Self-Assessment workflow. Built first in the F5 → F4 → F6 sequence due to its high-risk LLM dependency, the SKILL.md serves as the workflow orchestrator — sequencing calls to F6 deterministic scripts (input validation, artifact registry building, evidence mapping, citation validation) and performing its own reasoning between script calls: generating per-control narratives, assigning confidence tiers (HIGH/MEDIUM/LOW), and flagging gaps where zero evidence exists.

The LLM receives pre-processed evidence as file paths with content snippets, and produces two output documents:

- **`rcsa_control_narratives.md`** — Markdown with YAML front matter containing a summary table, per-control narratives, inline citations, confidence tiers, and gap flags
- **`validation_report.md`** — Citation resolution statistics

Output structure is defined inline within the SKILL.md since F4 templates are formalized afterward. The skill is framework-agnostic — controls are provided as input — with four demo controls for initial development: Access Control, Change Management, Data Quality, and Incident Handling. Ambiguous evidence is excluded and flagged for human review rather than included, prioritizing accuracy over coverage to prevent hallucination in compliance outputs.

---

## Technical Context

| Field | Value |
|-------|-------|
| **Language/Version** | N/A — SKILL.md is a natural language instruction file, not executable code |
| **Primary Dependencies** | F6 (Scripts — RCSA), F4 (Assets — RCSA), LLM agent runtime |
| **Storage** | N/A — produces Markdown files; persistence handled by agent platform and F6 |
| **Target Platform** | LLM agent runtime (framework-agnostic) |
| **Project Type** | Agent skill definition (natural language instruction file) |

### Testing

- **Priority 1**: Gap detection accuracy — zero false negatives on missing evidence
- LLM output quality assessed via rubric (narrative completeness, citation accuracy, confidence tier correctness)
- Citation validation handled by F6 scripts (`validation_report.md`)
- Edge case testing: zero-evidence controls, single weak artifact, ambiguous mappings, multi-control artifacts

### Performance Goals

| Metric | Target | Source |
|--------|--------|--------|
| Single control narrative | ≤ 30 seconds | NFR-001 |
| Full RCSA assessment (4 controls) | ≤ 120 seconds | NFR-002 |
| Per-LLM-call ceiling | ≤ 90 seconds | Constitution |
| Evidence file processing | ≤ 5 seconds per file | NFR-003 (F6 responsibility) |

### Constraints

- LLM context window must accommodate all evidence snippets per run; if evidence exceeds limits, chunking/prioritization is an F6 responsibility but F5 must handle partial input gracefully
- Anti-hallucination: exclude ambiguous evidence, flag for human review
- Gap flags must have zero false negatives

### Scale/Scope

- **Demo**: 4 controls, small codebase
- **Production**: NEEDS CLARIFICATION — Synchrony's control count and codebase size unknown; framework-agnostic design ensures adaptability

---

## Constitution Check

> **GATE**: Must pass before Phase 0 research. Re-check after Phase 1 design.

| Gate | Status | Evidence |
|------|--------|----------|
| Max 3 projects per repo | ✅ PASS | F5 is one feature within the existing repo structure |
| No unnecessary abstractions | ✅ PASS | SKILL.md is a single flat file — no abstraction layers |
| Each LLM call ≤ 90 seconds | ✅ PASS | NFR-001 targets 30s per control; **risk noted** — monitor multi-pass reasoning against 90s ceiling |
| Framework-agnostic design | ✅ PASS | Controls provided as input, no framework hardcoded |
| F5 → F4 → F6 build order | ✅ PASS | F5 built first; output structure defined inline for F4 to formalize later |
| Deterministic logic in scripts only | ✅ PASS | SKILL.md orchestrates F6 scripts for mechanical work; LLM handles reasoning only |
| Anti-hallucination guardrails | ✅ PASS | Ambiguous evidence excluded, gap flags zero false negatives, citation validation via F6 |

**No violations. One risk flagged for monitoring.**

---

## Project Structure

### Documentation (this feature)

```text
specs/005-skill-rcsa/
├── plan.md              # This file
├── research.md          # Phase 0 output (interview decisions)
├── data-model.md        # Phase 1 output (input/output schemas)
├── quickstart.md        # Phase 1 output (how to run/test the skill)
├── contracts/           # Phase 1 output (F5↔F4, F5↔F6 interfaces)
└── tasks.md             # Phase 2 output (implementation tasks)
```

### Source Code (repository root)

```text
skills/
└── rcsa/
    ├── SKILL.md                    # The agent skill file (primary deliverable)
    └── examples/
        ├── sample_input.yaml       # Demo evidence input (4 controls)
        └── expected_output/
            ├── rcsa_control_narratives.md
            └── validation_report.md
```

**Structure Decision**: Single flat skill directory. No `src/`, `models/`, or `services/` needed — F5 is a natural language file, not a code project. The `examples/` directory provides testable reference inputs and expected outputs for validation during development.

---

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified.**

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|--------------------------------------|
| *None* | — | — |

No complexity justifications required. One risk was flagged (90s per-call ceiling under multi-pass reasoning) but it is not a violation — it is a monitoring item.
