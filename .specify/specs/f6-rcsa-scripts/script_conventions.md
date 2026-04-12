# Script Conventions: F6 — RCSA Scripts (Deterministic Python Pipeline)

**Feature**: F6 — RCSA Scripts | **Date**: 2026-04-12 | **Status**: Draft

---

## Purpose

This document defines the shared conventions that all 6 F6 scripts follow. Each script implements these patterns independently (no shared module). This is the single source of truth — if a convention changes here, all 6 scripts must be updated to match.

---

## 1. Exit Code Conventions

All scripts use the same exit codes. F5's SKILL.md interprets these to decide pipeline flow.

| Exit Code | Meaning | When Used |
|---|---|---|
| `0` | Script completed successfully | Normal execution — check `status` field in JSON output for result |
| `1` | Unexpected error (bug) | Unhandled exception, missing dependency, programming error |

**Important**: Exit code `0` does NOT mean "everything passed." It means the script ran without crashing. The JSON output's `status` field carries the actual result:

| `status` Value | Meaning | Exit Code |
|---|---|---|
| `"pass"` | Validation passed (Step 1 only) | `0` |
| `"fail"` | Validation failed (Step 1 only) — F5 stops pipeline | `0` |
| `"success"` | Script produced valid output (Steps 2–6) | `0` |
| `"degraded"` | LLM failed but pipeline continues (Step 4b only) | `0` |

**Rationale**: Separating "script crashed" (exit 1) from "script ran but found problems" (exit 0 + status) lets F5 distinguish between bugs and expected failure modes.

---

## 2. Error Message Format

All error messages follow this format:

```
[F6-{SCRIPT}] {ERROR_TYPE}: {message}
```

**Components**:

| Component | Format | Example |
|---|---|---|
| `F6` | Fixed prefix — identifies the feature | `F6` |
| `{SCRIPT}` | Uppercase script identifier | `VALIDATE`, `REGISTRY`, `MAP`, `LLM`, `CITATIONS`, `OUTPUT` |
| `{ERROR_TYPE}` | Category of error | `FILE_NOT_FOUND`, `SCHEMA_ERROR`, `PARSE_ERROR`, `WRITE_ERROR` |
| `{message}` | Human-readable description | `Missing required field: controls[2].objective` |

**Script identifiers**:

| Script | Identifier |
|---|---|
| `validate_input.py` | `VALIDATE` |
| `build_registry.py` | `REGISTRY` |
| `map_evidence.py` | `MAP` |
| `orchestrate_llm.py` | `LLM` |
| `validate_citations.py` | `CITATIONS` |
| `write_output.py` | `OUTPUT` |

**Error type vocabulary** (use only these):

| Error Type | When Used |
|---|---|
| `FILE_NOT_FOUND` | Expected input file does not exist |
| `SCHEMA_ERROR` | JSON structure does not match expected schema |
| `PARSE_ERROR` | Cannot parse input (invalid JSON, malformed LLM response) |
| `VALIDATION_ERROR` | Data fails a validation rule (duplicate IDs, invalid references) |
| `WRITE_ERROR` | Cannot write output file to disk |
| `LLM_ERROR` | LLM-related failure (empty response, timeout, malformed output) |
| `INTERNAL_ERROR` | Unexpected bug — should not happen in normal operation |

**Examples**:

```
[F6-VALIDATE] FILE_NOT_FOUND: sample_control_library.json not found at .cursor/skills/rcsa/data/sample_control_library.json
[F6-VALIDATE] SCHEMA_ERROR: Missing required field: controls[2].objective
[F6-VALIDATE] VALIDATION_ERROR: Duplicate control ID 'AC' found at controls[0] and controls[3]
[F6-REGISTRY] PARSE_ERROR: sample_artifact_index.json is not valid JSON — Expecting ',' delimiter: line 15 column 3
[F6-LLM] LLM_ERROR: Empty response received — entering degraded mode
[F6-CITATIONS] VALIDATION_ERROR: Citation 'nonexistent/file.py' does not resolve to any artifact or test in the registry
[F6-OUTPUT] WRITE_ERROR: Cannot write to .cursor/skills/rcsa/output/rcsa_control_narratives.md — Permission denied
```

---

## 3. Logging Conventions

### Output channels

| Channel | Used For |
|---|---|
| **Return value** (JSON) | Structured data passed to F5 agent runtime via tool-calling |
| **stderr** | Human-readable log messages for debugging |

Scripts return structured JSON data to the F5 agent runtime via Cursor's tool-calling mechanism (per RQ-001 decision). Stderr is used for logging only — it is not part of the data pipeline.

### Stderr log format

```
[F6-{SCRIPT}] {LEVEL}: {message}
```

**Log levels**:

| Level | When Used | Example |
|---|---|---|
| `INFO` | Normal operation milestones | `[F6-VALIDATE] INFO: Validating 3 input files` |
| `WARN` | Non-fatal issues | `[F6-MAP] WARN: Control DQ has 0 mapped tests` |
| `ERROR` | Errors (also included in JSON output) | `[F6-VALIDATE] ERROR: FILE_NOT_FOUND: sample_control_library.json not found` |

**Rules**:
- Every script logs `INFO: Starting {script_name}` at the beginning
- Every script logs `INFO: Completed — status: {status}` at the end
- Errors are logged to stderr AND included in the JSON output's `errors` array
- No logging to stdout — stdout is reserved (even though primary data flow uses tool-calling return values, stdout must remain clean)

---

## 4. Supported Control Codes

The canonical list of control IDs for the demo:

| Control ID | Control Name |
|---|---|
| `AC` | Access Control |
| `CM` | Change Management |
| `DQ` | Data Quality |
| `IH` | Incident Handling |

**Validation rules**:
- Control IDs must match the regex pattern: `^[A-Z]{2,4}$`
- The 4 IDs above are the demo dataset — scripts do NOT hardcode these values
- Scripts validate the **format** (regex), not the **specific values**
- If F4 adds a 5th control that matches the regex, all scripts process it without code changes
- `validate_input.py` rejects any control ID that does not match the regex
- `validate_input.py` also rejects any `tests[].controls_relevant` value that does not match a `controls[].id` in the control library

---

## 5. Data Flow Conventions

**Mechanism**: Agent tool-calling (RQ-001 — decided)

F5's SKILL.md invokes each F6 script as a tool via the Cursor agent runtime. The agent holds each script's return value in memory and passes it as input to the next script. There is no stdout piping and no temp file handoff between scripts.

**Invocation sequence**:

| Step | Script | Receives From | Returns To |
|---|---|---|---|
| 1 | `validate_input.py` | File paths (strings) | `ValidationResult` → F5 |
| 2 | `build_registry.py` | File paths (strings) | `ArtifactRegistry` → F5 |
| 3 | `map_evidence.py` | `ArtifactRegistry` + control library data | `MappedEvidence` → F5 |
| 4 | `orchestrate_llm.py` (format mode) | `MappedEvidence` | Formatted prompt string → F5 |
| — | F5 invokes LLM | Prompt from Step 4 | Raw LLM response |
| 4b | `orchestrate_llm.py` (parse mode) | Raw LLM response + `MappedEvidence` | `LLMResponse` → F5 |
| 5 | `validate_citations.py` | `LLMResponse` + `ArtifactRegistry` | `CitationValidationResult` → F5 |
| 6 | `write_output.py` | `LLMResponse` + `CitationValidationResult` + `MappedEvidence` | Output file paths → F5 |

**Rules**:
- Each script receives its input as function arguments (JSON objects or strings)
- Each script returns its output as a JSON object (or string for Step 4)
- Scripts do NOT read from stdin or write structured data to stdout
- Scripts do NOT write intermediate files — all data stays in the agent's memory
- The only files written to disk are the final outputs in Step 6

**`orchestrate_llm.py` dual-mode convention**:

This script is called twice with different modes:
- **Format mode** (`--mode format`): Takes `MappedEvidence`, returns a formatted prompt string
- **Parse mode** (`--mode parse`): Takes raw LLM response + `MappedEvidence`, returns `LLMResponse`

The `--mode` flag determines which function executes. Both functions live in the same script because they share knowledge of the expected LLM response structure.

---

## 6. Citation Format

**Format**: `[file_path — description]` (RQ-002 — decided)

| Component | Description | Example |
|---|---|---|
| `file_path` | Relative path from repository root, matching `artifacts[].file_path` or `tests[].file_path` | `auth/oauth_config.yaml` |
| ` — ` | Space, em dash (U+2014), space — this is the delimiter | ` — ` |
| `description` | Human-readable description of the artifact | `OAuth2 provider configuration` |

**Regex pattern for parsing**:

```
\[([^\]]+?)\s—\s([^\]]+?)\]
```

- Group 1: `file_path` (validated against `registry.file_path_index`)
- Group 2: `description` (logged but not validated)

**Validation rules** (applied by `validate_citations.py`):
- Extract all citations matching the regex from the LLM's narrative text
- Look up each `file_path` in `registry.file_path_index`
- If found → citation is **resolved** (maps to an artifact or test ID)
- If not found → citation is **unresolved** (hallucinated by the LLM)
- Unresolved citations are reported in `validation_report.md` — they do NOT stop the pipeline

**Important**: The delimiter is an em dash (`—`, U+2014), NOT a hyphen (`-`) or en dash (`–`). All three features (F4, F5, F6) are aligned on this.

---

## 7. Confidence Tier Definitions

**Owner**: F5 (the LLM assigns tiers using F5's rubric). F6 extracts and displays them.

| Tier | Meaning | Assigned When |
|---|---|---|
| `HIGH` | Strong evidence across all evidence types | Multiple artifacts + tests covering all evidence types for the control |
| `MEDIUM` | Partial evidence — some types covered | Some evidence types have artifacts, others don't |
| `LOW` | Weak evidence — minimal coverage | Few artifacts, few or no tests |
| `GAP` | No evidence found | Zero artifacts AND zero tests mapped to the control |
| `UNAVAILABLE` | LLM did not assign a tier | F6 default when the confidence line is missing from the LLM response, or when the LLM is unavailable (degraded mode) |

**F6's responsibilities**:
- `map_evidence.py` sets `gap_flag: true` on controls with zero artifacts AND zero tests — this is a deterministic check independent of the LLM
- `orchestrate_llm.py` (parse mode) extracts the tier from the `**Confidence**: {TIER}` line in each control section
- If the confidence line is missing or contains an unrecognized value → default to `UNAVAILABLE`
- `write_output.py` includes the tier in the YAML front matter and summary table
- F6 does NOT calculate or override the LLM's tier assignment (except defaulting to `UNAVAILABLE` when missing)

---

## 8. Output Directory Convention

**Output path**: `.cursor/skills/rcsa/output/`

| File | Description |
|---|---|
| `rcsa_control_narratives.md` | Final narrative document with YAML front matter, summary table, and per-control narratives |
| `validation_report.md` | Citation resolution report and evidence coverage analysis |

**Rules**:
- `write_output.py` creates the `output/` directory at runtime if it does not exist
- `write_output.py` overwrites existing files without prompting (demo behavior)
- File encoding: UTF-8
- Line endings: LF (Unix-style)
- If the directory cannot be created or files cannot be written, `write_output.py` exits with code `1` and logs a `WRITE_ERROR`

---

## 9. Common Python Patterns

Each script implements these patterns independently. No shared imports.

### 9.1 JSON reading

```python
import json
import sys

def read_json_file(file_path):
    try:
        with open(file_path, 'r', encoding='utf-8') as f:
            return json.load(f)
    except FileNotFoundError:
        print(f"[F6-SCRIPTNAME] ERROR: FILE_NOT_FOUND: {file_path}", file=sys.stderr)
        sys.exit(1)
    except json.JSONDecodeError as e:
        print(f"[F6-SCRIPTNAME] ERROR: PARSE_ERROR: {file_path} is not valid JSON — {e}", file=sys.stderr)
        sys.exit(1)
```

Each script replaces `SCRIPTNAME` with its own identifier (e.g., `VALIDATE`, `REGISTRY`).

### 9.2 Stderr logging

```python
import sys

def log(script_id, level, message):
    print(f"[F6-{script_id}] {level}: {message}", file=sys.stderr)
```

Each script defines this function locally or inlines the pattern.

### 9.3 Script entry point

```python
def main():
    log("SCRIPTNAME", "INFO", "Starting script_name.py")
    # ... script logic ...
    log("SCRIPTNAME", "INFO", f"Completed — status: {result['status']}")
    return result

if __name__ == "__main__":
    main()
```

---

## 10. What This Document Does NOT Cover

| Topic | Where It Lives |
|---|---|
| Data structure schemas | `data-model.md` |
| F4 input file contracts | `F6_F4_contracts.md` |
| F5 invocation and LLM contracts | `F6_F5_contracts.md` |
| Implementation task list | `tasks.md` |
| Architecture and phasing | `plan.md` |
| Research decisions | `research.md` |

---

*This document defines the conventions all 6 F6 scripts follow. It is a reference for implementers. If a convention changes, all 6 scripts must be updated to match. Changes to conventions that affect F5 (exit codes, data flow, citation format) require review of the F6↔F5 contract.*
