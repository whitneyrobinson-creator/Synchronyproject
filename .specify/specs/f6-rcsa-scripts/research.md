# Research: F6 — RCSA Scripts (Deterministic Python Pipeline)

**Feature**: F6 — RCSA Scripts | **Date**: 2026-04-12 | **Status**: All questions resolved

---

## Purpose

This document captures the research questions identified during Phase 0 planning, the options considered, and the final decisions. All questions were resolved through review of F4's locked deliverables and F5's SKILL.md.

---

## RQ-001: How does data flow between F6 scripts via the SKILL.md agent?

**Context**: Each F6 script is invoked individually by the SKILL.md agent. How does output from one script get passed to the next?

| Option | Mechanism | Pros | Cons |
|---|---|---|---|
| A — stdout/stdin piping | Script prints JSON to stdout, agent captures and pipes to next script | Simple, no temp files | Agent must support piping |
| B — Temp file handoff | Script writes JSON to a known temp file path, next script reads it | Inspectable for debugging | Creates intermediate files on disk |
| **C — Agent tool-calling** | **SKILL.md invokes each script as a tool via Cursor agent runtime. Agent holds return values in memory and passes them as arguments to the next tool call.** | **Native to Cursor, truly in-memory, no file I/O between scripts** | **Coupled to Cursor runtime** |

**Decision**: ✅ **Option C — Agent tool-calling**

**Rationale**: F5's SKILL.md defines a 6-step workflow where each step invokes an F6 script as a tool. The Cursor agent runtime natively supports tool-calling — the agent holds each script's return value in context and passes it to the next script. No stdout piping, no temp files. This is how Cursor agent skills are designed to work.

**Source**: F5 SKILL.md — step-by-step workflow definition.

---

## RQ-002: What is the exact citation format F5 will produce?

**Context**: `validate_citations.py` needs to parse citations from the LLM's output. What format will they be in?

| Format | Example | Regex Pattern |
|---|---|---|
| Bracket-ID | `[ART-001]` | `\[ART-\d{3}\]` |
| Bracket-ID with name | `[ART-001: scan_report.pdf]` | `\[ART-\d{3}:\s*.+?\]` |
| **File path with description** | **`[scan_results/vuln_report.pdf — Q4 vulnerability scan results]`** | **`\[.+?\s—\s.+?\]`** |

**Decision**: ✅ **`[file_path — description]` format**

**Rationale**: This is the format defined in F4's `citation_format.md` (updated to align with F5). F5's SKILL.md instructs the LLM to use this format. All three features (F4, F5, F6) are now aligned.

**Source**: F4 `citation_format.md` (updated 2026-04-12), F5 SKILL.md.

**Impact on F6**: `validate_citations.py` will use a regex pattern matching `[...—...]` to extract citations, then resolve the file path portion against the artifact registry.

---

## RQ-003: What does the LLM response structure look like?

**Context**: `orchestrate_llm.py` needs to parse the LLM's response. What structure should F6 expect?

| Option | Structure | Parsing Complexity |
|---|---|---|
| A — Free-form Markdown | LLM writes prose with headings per control | High — fragile heading parsing |
| **B — Structured Markdown with delimiters** | **LLM wraps each control section in known delimiters** | **Low — reliable delimiter parsing** |
| C — JSON response | LLM returns JSON with keys per control | Medium — LLM may produce invalid JSON |

**Decision**: ✅ **Option B — Structured Markdown with delimiters**

**Rationale**: F5's SKILL.md instructs the LLM to produce narrative output organized by control, with clear section boundaries. The SKILL.md defines the expected output structure that `orchestrate_llm.py` can reliably parse. The LLM writes natural prose within a predictable structure.

**Source**: F5 SKILL.md — output format instructions.

**Impact on F6**: `orchestrate_llm.py` parses the LLM response by splitting on control section boundaries. If a section is missing or malformed, graceful degradation kicks in for that control only — other controls are unaffected.

---

## RQ-004: How does F6 determine confidence tiers?

**Context**: The output includes confidence tiers per control. What determines the tier, and who calculates it?

| Option | Who Calculates | How |
|---|---|---|
| A — F6 calculates deterministically | `map_evidence.py` or `validate_citations.py` | Based on artifact/test counts and citation validity |
| **B — LLM assigns during narrative generation** | **F5 (the LLM, guided by SKILL.md)** | **LLM evaluates evidence strength and assigns tier using F5's rubric** |

**Decision**: ✅ **Option B — LLM assigns confidence tiers**

**Rationale**: F5's SKILL.md includes a confidence tier rubric (HIGH / MEDIUM / LOW / GAP) and instructs the LLM to assign a tier to each control during Step 4 (narrative generation). F6 does not calculate tiers — it receives them as part of the LLM's response and includes them in the output.

**Source**: F5 SKILL.md — confidence tier rubric and Step 4 instructions.

**Impact on F6**:
- `map_evidence.py` still flags controls with no evidence (`gap_flag: true`) — this is a deterministic check independent of the LLM
- `orchestrate_llm.py` extracts the LLM-assigned tier from the response
- `write_output.py` includes the tier in the YAML front matter of each control section
- If the LLM is unavailable (graceful degradation), F6 assigns `"UNAVAILABLE"` as the tier — it does not guess

---

## RQ-005: What are the exact F4 schema structures?

**Context**: `validate_input.py` validates against F4's JSON schemas. Are the schemas finalized?

**Decision**: ✅ **Schemas are locked in F4**

**Rationale**: F4 is complete. The following files define the schemas:

| File | Defines |
|---|---|
| `control_library.schema.json` | Control library structure. Control IDs match `^[A-Z]{2,4}$` (e.g., AC, CM, DQ, IH) |
| `sample_control_library.json` | Sample data — 4 controls |
| `sample_artifact_index.json` | Sample data — 10-20 artifacts with file paths and metadata |
| `sample_test_catalog.json` | Sample data — 5-10 tests with `controls_relevant[]` arrays |

**Key details for F6**:
- Control IDs are **short uppercase strings**: `AC`, `CM`, `DQ`, `IH`
- Control IDs are enforced by schema regex: `^[A-Z]{2,4}$`
- Test-to-control mapping uses `tests[].controls_relevant[]` containing these short IDs
- All files are JSON, UTF-8 encoded

**Source**: F4 locked deliverables.

**Impact on F6**: `validate_input.py` validates against these schemas. `build_registry.py` and `map_evidence.py` use the short control ID format for all matching logic.

---

## Cross-Feature Alignment Summary

All research questions were resolved by examining F4 and F5's locked deliverables. The following alignment was confirmed or corrected during planning:

| Item | F4 | F5 | F6 | Status |
|---|---|---|---|---|
| Citation format | `[file_path — description]` | `[file_path — description]` | Parses this pattern | ✅ Aligned |
| Control IDs | `AC`, `CM`, `DQ`, `IH` | Updated to match F4 | Matches on short IDs | ✅ Aligned |
| Data format | JSON files on disk | Reads JSON, passes to LLM | Reads JSON via `json.load()` | ✅ Aligned |
| Script invocation | N/A | SKILL.md tool-calling workflow | Scripts return data to agent | ✅ Aligned |
| Confidence tiers | N/A | LLM assigns using rubric | F6 extracts, does not calculate | ✅ Aligned |
| Graceful degradation | N/A | SKILL.md defines fallback | F6 produces raw evidence output | ✅ Aligned |

---

## Research Gate: PASS

All 5 research questions resolved. Zero open items. Phase 1 design can proceed.
