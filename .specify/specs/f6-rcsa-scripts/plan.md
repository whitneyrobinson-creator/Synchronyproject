# Implementation Plan: F6 — RCSA Scripts (Deterministic Python Pipeline)

**Branch**: `f6-rcsa-scripts` | **Date**: 2026-04-07 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/f6-rcsa-scripts/spec.md`

---

## Summary

Build the deterministic Python pipeline that supports the RCSA Control Narrative Generation skill. The scripts handle everything outside the LLM reasoning step: input validation (JSON existence, validity, schema conformance), artifact registry building, evidence-to-control mapping using the control library, LLM invocation orchestration with retry logic and graceful degradation, citation validation against the artifact registry to detect hallucinations, and output file writing (`rcsa_control_narratives.md`, `validation_report.md`). The scripts consume the output structure defined by F5 (SKILL.md) and produce inputs that F4 (Assets) will later formalize into reusable templates and sample data.

---

## Technical Context

**Language/Version**: Python 3.11 (Constitution Tech Stack requirement)  
**Primary Dependencies**: `json` (stdlib — input parsing), `os`/`pathlib` (stdlib — file I/O), `logging` (stdlib — audit trail), `re` (stdlib — citation parsing), `python-dotenv` (env var management for API keys), `anthropic` (Claude API client for LLM invocation)  
**Storage**: Local filesystem only — no database, no cloud storage (Constitution constraint). Input files read from project directory; output files written to local output directory as Markdown.  
**Testing**: `pytest` — unit tests for each pipeline module (validator, registry builder, evidence mapper, citation validator, output writer); integration test for full pipeline end-to-end using sample data; mocked LLM responses for orchestrator testing  
**Target Platform**: Local developer machine (macOS/Linux/Windows); Claude Agent Skills runtime (Cursor IDE integration)

**Project Type**: CLI / library — Python scripts invoked by the Claude Agent Skills runtime or run standalone for testing  
**Performance Goals**: Full pipeline completes in < 60 seconds for demo-scale inputs; input validation completes in < 5 seconds  
**Constraints**: No hardcoded API keys — must use environment variables (Constitution security requirement); must function without LLM (graceful degradation — Constitution Principle 2); all processing uses local files only  
**Scale/Scope**: Demo-scale repositories; 4 compliance controls (AC, CM, DQ, IH); designed for the May 7th demo, not enterprise-scale

---

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

**Status: PASS**

| Rule | Plan Compliance |
|------|-----------------|
| Principle 1: Accuracy Over Speed — never imply compliance without proof | ✅ Citation validation (FR-008) checks every citation against the artifact registry; hallucinated citations are flagged, not silently accepted |
| Principle 2: Graceful Degradation — scripts must function without LLM | ✅ FR-007 requires graceful degradation output on all LLM failures; pipeline never crashes due to LLM unavailability |
| Section 3: No permanent data storage; outputs stored locally as Markdown | ✅ FR-012 restricts storage to local filesystem; output files are Markdown only |
| Section 4: All input validation handled by deterministic scripts before LLM | ✅ FR-001 validates all inputs before any LLM invocation; malformed data never reaches the LLM |
| Tech Stack: Python 3.11, Claude Sonnet | ✅ FR-011 requires Python 3.11; LLM orchestrator targets Claude Sonnet via Anthropic API |
| Security: No hardcoded secrets | ✅ FR-014 requires API keys in environment variables (.env file); 0 hardcoded secrets in source |
| Audit Trail: Log all processing steps | ✅ FR-013 requires logging of all pipeline steps and errors for debugging and audit purposes |

---

## Project Structure

### Documentation (this feature)

```text
specs/f6-rcsa-scripts/
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
└── tools/
    ├── validate_inputs.py          # FR-001: Input validation (existence, JSON, schema)
    ├── build_artifact_registry.py  # FR-002: Build structured artifact index
    ├── map_evidence.py             # FR-003, FR-004: Map artifacts to controls using control library
    ├── orchestrate_llm.py          # FR-005, FR-006, FR-007: LLM invocation, retries, graceful degradation
    ├── validate_citations.py       # FR-008, FR-009: Citation validation against registry
    ├── write_output.py             # FR-010: Write Markdown output files
    └── pipeline.py                 # Main entry point — orchestrates all steps in sequence

tests/
├── unit/
│   ├── test_validate_inputs.py
│   ├── test_build_artifact_registry.py
│   ├── test_map_evidence.py
│   ├── test_orchestrate_llm.py
│   ├── test_validate_citations.py
│   └── test_write_output.py
└── integration/
    └── test_full_pipeline.py       # End-to-end test with sample data and mocked LLM
```

**Structure Decision**: Single project with modular scripts. Each pipeline step is a separate Python module for independent testing and clear separation of concerns. `pipeline.py` is the main entry point that calls each module in sequence. Tests mirror the source structure with unit tests per module and one integration test for the full pipeline. This aligns with the Constitution's simplicity principle — no unnecessary abstractions or frameworks.

---

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

*No violations. Constitution Check passed with no exceptions.*
