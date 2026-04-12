# Quickstart: F6 — RCSA Scripts (Deterministic Python Pipeline)

**Feature**: F6 — RCSA Scripts | **Date**: 2026-04-12

---

## Overview

F6 consists of 6 Python scripts that run in sequence — 3 before the LLM reasons and 3 after. There are two ways to run them:

1. **Agent workflow** — The SKILL.md in Cursor invokes each script as a tool call. This is the normal production flow.
2. **Standalone workflow** — You run each script individually from the command line. This is for testing, debugging, and understanding what each script does.

Both workflows produce the same outputs: `rcsa_control_narratives.md` and `validation_report.md`.

---

## Prerequisites

- **Python 3.11** installed and available on PATH
- **No pip installs required** — all scripts use Python standard library only
- **F4 sample data files** available at known paths:
  - `sample_control_library.json`
  - `sample_artifact_index.json`
  - `sample_test_catalog.json`

### Verify Python version

```bash
python --version
# Expected: Python 3.11.x
```

---

## File Locations

```
.cursor/skills/rcsa/
├── SKILL.md                          # F5 — agent orchestration instructions
├── scripts/
│   ├── validate_input.py             # Step 1: Validate JSON inputs
│   ├── build_registry.py             # Step 2: Build artifact registry
│   ├── map_evidence.py               # Step 3: Map evidence to controls
│   ├── orchestrate_llm.py            # Step 4: Format for LLM + parse response
│   ├── validate_citations.py         # Step 5: Validate citations
│   └── write_output.py               # Step 6: Write output files
├── data/                             # F4 — sample input data
│   ├── sample_control_library.json
│   ├── sample_artifact_index.json
│   └── sample_test_catalog.json
└── output/                           # Generated at runtime (gitignored)
    ├── rcsa_control_narratives.md
    └── validation_report.md
```

---

## Workflow 1: Agent Invocation (Normal Use)

This is how the pipeline runs in production via Cursor.

### Step 1: Open Cursor and invoke the skill

The SKILL.md defines the full workflow. When invoked, the agent:

1. Calls `validate_input.py` with paths to the 3 JSON files
2. If validation passes, calls `build_registry.py` with the validated data
3. Calls `map_evidence.py` with the registry and control library
4. Calls `orchestrate_llm.py` with the mapped evidence — this is where the agent performs LLM reasoning
5. Calls `validate_citations.py` with the LLM response and the registry
6. Calls `write_output.py` with all results to produce the final Markdown files

### What to expect

- The agent holds each script's return value in memory and passes it to the next script
- No intermediate files are written between scripts
- Total runtime: < 60 seconds for deterministic steps + LLM thinking time
- Output files appear in `.cursor/skills/rcsa/output/`

### If something goes wrong

- If `validate_input.py` fails → the agent stops and reports validation errors
- If the LLM is unavailable → `orchestrate_llm.py` returns a degraded response → pipeline continues in raw evidence mode
- Check Cursor's output panel for logged errors from each script

---

## Workflow 2: Standalone Script Invocation (Testing & Debugging)

Run each script individually from the command line. This lets you inspect the output of each step before moving to the next.

### Setup

Navigate to the scripts directory:

```bash
cd .cursor/skills/rcsa/scripts
```

All scripts read from stdin (JSON) and write to stdout (JSON), unless otherwise noted. This makes them pipeable for testing.

---

### Step 1: Validate Inputs

```bash
python validate_input.py \
  --control-library ../data/sample_control_library.json \
  --artifact-index ../data/sample_artifact_index.json \
  --test-catalog ../data/sample_test_catalog.json
```

**Expected output (success):**

```json
{
  "status": "pass",
  "files_validated": [
    {"file": "control_library", "status": "pass", "errors": []},
    {"file": "artifact_index", "status": "pass", "errors": []},
    {"file": "test_catalog", "status": "pass", "errors": []}
  ]
}
```

**Expected output (failure):**

```json
{
  "status": "fail",
  "files_validated": [
    {"file": "control_library", "status": "fail", "errors": ["Missing required field: controls[2].objective"]}
  ]
}
```

**What to check:**
- Exit code 0 = pass, exit code 1 = fail
- If fail, read the `errors` array — it tells you exactly what's wrong and where

**Stop here if validation fails.** Fix the input files before proceeding.

---

### Step 2: Build Registry

```bash
python build_registry.py \
  --control-library ../data/sample_control_library.json \
  --artifact-index ../data/sample_artifact_index.json \
  --test-catalog ../data/sample_test_catalog.json
```

**Expected output (abbreviated):**

```json
{
  "registry": {
    "artifacts": {
      "ART-001": {"id": "ART-001", "file_path": "auth/oauth_config.yaml", "...": "..."},
      "ART-002": {"id": "ART-002", "file_path": "auth/mfa_settings.json", "...": "..."}
    },
    "tests": {
      "TST-001": {"id": "TST-001", "file_path": "tests/auth/test_rbac_permissions.py", "...": "..."}
    },
    "file_path_index": {
      "auth/oauth_config.yaml": "ART-001",
      "tests/auth/test_rbac_permissions.py": "TST-001"
    },
    "stats": {
      "total_artifacts": 15,
      "total_tests": 8,
      "total_entries": 23
    }
  }
}
```

**What to check:**
- `stats.total_artifacts` = 15 (matches sample data)
- `stats.total_tests` = 8 (matches sample data)
- `file_path_index` has 23 entries (15 artifacts + 8 tests)
- Every artifact and test from the input files appears in the registry

**Save output for next step (optional for debugging):**

```bash
python build_registry.py \
  --control-library ../data/sample_control_library.json \
  --artifact-index ../data/sample_artifact_index.json \
  --test-catalog ../data/sample_test_catalog.json \
  > /tmp/registry.json
```

---

### Step 3: Map Evidence

```bash
python map_evidence.py \
  --registry /tmp/registry.json \
  --control-library ../data/sample_control_library.json
```

**Expected output (abbreviated):**

```json
{
  "mappings": [
    {
      "control_id": "AC",
      "mapped_artifacts": ["ART-001", "ART-002", "ART-003", "ART-004", "ART-005"],
      "mapped_tests": ["TST-001", "TST-002", "TST-003"],
      "gap_flag": false,
      "artifact_count": 5,
      "test_count": 3
    },
    {
      "control_id": "CM",
      "artifact_count": 6,
      "test_count": 3,
      "gap_flag": false
    },
    {
      "control_id": "DQ",
      "artifact_count": 2,
      "test_count": 1,
      "gap_flag": false
    },
    {
      "control_id": "IH",
      "artifact_count": 2,
      "test_count": 1,
      "gap_flag": false
    }
  ],
  "summary": {
    "total_controls": 4,
    "controls_with_evidence": 4,
    "controls_with_gaps": 0
  }
}
```

**What to check:**
- All 4 controls appear in `mappings`
- AC: 5 artifacts, 3 tests
- CM: 6 artifacts, 3 tests
- DQ: 2 artifacts, 1 test
- IH: 2 artifacts, 1 test
- `controls_with_gaps` = 0 (all controls have at least some evidence)
- Check `evidence_type_coverage` — DQ is missing `data_transform_test`, IH is missing `incident_response_plan` and `postmortem_record`

**Save output for next step:**

```bash
python map_evidence.py \
  --registry /tmp/registry.json \
  --control-library ../data/sample_control_library.json \
  > /tmp/mapped_evidence.json
```

---

### Step 4: Orchestrate LLM

In standalone mode, this script formats the mapped evidence for the LLM but **does not call the LLM**. Instead, you provide a mock LLM response.

**Format evidence (see what the LLM would receive):**

```bash
python orchestrate_llm.py \
  --mapped-evidence /tmp/mapped_evidence.json \
  --mode format
```

This outputs the formatted prompt that would be sent to the LLM. Useful for reviewing what the LLM sees.

**Parse a mock LLM response:**

```bash
python orchestrate_llm.py \
  --mapped-evidence /tmp/mapped_evidence.json \
  --llm-response tests/mocks/mock_llm_response_success.md \
  --mode parse
```

**Test graceful degradation:**

```bash
python orchestrate_llm.py \
  --mapped-evidence /tmp/mapped_evidence.json \
  --llm-response tests/mocks/mock_llm_response_empty.md \
  --mode parse
```

**Expected output (degraded):**

```json
{
  "status": "degraded",
  "reason": "LLM unavailable — raw evidence provided without narratives",
  "narratives": {},
  "mapped_evidence_passthrough": {"...": "full mapped evidence"}
}
```

**What to check:**
- `--mode format`: The output is a well-structured prompt with all 4 controls and their evidence
- `--mode parse` with success mock: All 4 controls have narratives with citations
- `--mode parse` with empty mock: Status is `"degraded"`, no crash, mapped evidence passed through

**Save output for next step:**

```bash
python orchestrate_llm.py \
  --mapped-evidence /tmp/mapped_evidence.json \
  --llm-response tests/mocks/mock_llm_response_success.md \
  --mode parse \
  > /tmp/llm_response.json
```

---

### Step 5: Validate Citations

```bash
python validate_citations.py \
  --llm-response /tmp/llm_response.json \
  --registry /tmp/registry.json
```

**Expected output (all citations valid):**

```json
{
  "total_citations": 15,
  "resolved": 15,
  "unresolved": 0,
  "resolution_rate": 1.0,
  "citations": [
    {
      "citation_text": "auth/oauth_config.yaml — OAuth2 provider configuration",
      "file_path_extracted": "auth/oauth_config.yaml",
      "resolved_to": "ART-001",
      "resolved": true,
      "control_id": "AC"
    }
  ],
  "unresolved_list": []
}
```

**What to check:**
- `resolution_rate` = 1.0 for the success mock (all citations should resolve)
- If using a mock with hallucinated citations, `unresolved_list` should contain them
- Every citation's `file_path_extracted` should match an entry in `registry.file_path_index`

**Save output for next step:**

```bash
python validate_citations.py \
  --llm-response /tmp/llm_response.json \
  --registry /tmp/registry.json \
  > /tmp/citation_validation.json
```

---

### Step 6: Write Output

```bash
python write_output.py \
  --llm-response /tmp/llm_response.json \
  --citation-validation /tmp/citation_validation.json \
  --mapped-evidence /tmp/mapped_evidence.json \
  --output-dir ../output
```

**Expected output:**

```
Written: .cursor/skills/rcsa/output/rcsa_control_narratives.md
Written: .cursor/skills/rcsa/output/validation_report.md
```

**What to check:**
- Both files exist in the output directory
- Open `rcsa_control_narratives.md`:
  - YAML front matter is present and valid
  - Summary table shows all 4 controls with confidence tiers
  - Each control section has a narrative with inline citations
  - No raw `{{placeholder}}` strings remain
- Open `validation_report.md`:
  - Citation resolution table is populated
  - Evidence coverage table shows which evidence types are covered/missing
  - Resolution rate matches what `validate_citations.py` reported

---

## Full Standalone Pipeline (One Command)

For convenience, you can pipe all steps together. Note: this skips the LLM step and uses a mock response.

```bash
cd .cursor/skills/rcsa/scripts

# Validate → Build → Map → Parse Mock → Validate Citations → Write Output
python validate_input.py \
  --control-library ../data/sample_control_library.json \
  --artifact-index ../data/sample_artifact_index.json \
  --test-catalog ../data/sample_test_catalog.json \
&& python build_registry.py \
  --control-library ../data/sample_control_library.json \
  --artifact-index ../data/sample_artifact_index.json \
  --test-catalog ../data/sample_test_catalog.json \
  > /tmp/registry.json \
&& python map_evidence.py \
  --registry /tmp/registry.json \
  --control-library ../data/sample_control_library.json \
  > /tmp/mapped_evidence.json \
&& python orchestrate_llm.py \
  --mapped-evidence /tmp/mapped_evidence.json \
  --llm-response tests/mocks/mock_llm_response_success.md \
  --mode parse \
  > /tmp/llm_response.json \
&& python validate_citations.py \
  --llm-response /tmp/llm_response.json \
  --registry /tmp/registry.json \
  > /tmp/citation_validation.json \
&& python write_output.py \
  --llm-response /tmp/llm_response.json \
  --citation-validation /tmp/citation_validation.json \
  --mapped-evidence /tmp/mapped_evidence.json \
  --output-dir ../output
```

If any step fails, the `&&` chain stops and you see the error from the failing script.

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| `validate_input.py` fails with "file not found" | Wrong path to JSON files | Check `--control-library`, `--artifact-index`, `--test-catalog` paths |
| `validate_input.py` fails with "invalid control ID" | Control ID doesn't match `^[A-Z]{2,4}$` | Fix the control library JSON — IDs must be uppercase 2-4 chars |
| `validate_input.py` fails with "unknown control reference" | A test references a control ID not in the control library | Fix `controls_relevant` in the test catalog |
| `build_registry.py` shows wrong counts | Duplicate IDs in input files | Check for duplicate `ART-NNN` or `TST-NNN` entries |
| `map_evidence.py` shows unexpected gaps | Artifact descriptions don't match evidence type descriptions | Review the mapping logic — may need keyword adjustments |
| `orchestrate_llm.py` returns degraded | Mock file is empty or malformed | Check the mock file contents |
| `validate_citations.py` shows unresolved citations | LLM cited a file path not in the artifact index | Check the mock response — hallucinated citations are expected to be caught |
| `write_output.py` leaves `{{placeholder}}` in output | Template population logic missed a field | Check which placeholder wasn't populated — likely a missing key in the data |
| `ModuleNotFoundError` | Wrong Python version or missing stdlib module | Verify `python --version` is 3.11.x |

---

## Expected Demo Results

When run against the F4 sample data with a well-formed LLM response:

| Control | Artifacts | Tests | Expected Confidence | Notes |
|---|---|---|---|---|
| AC | 5 | 3 | HIGH | Full coverage across all 3 evidence types |
| CM | 6 | 3 | HIGH | Full coverage across all 3 evidence types |
| DQ | 2 | 1 | MEDIUM | Missing `data_transform_test` evidence type |
| IH | 2 | 1 | LOW | Missing `incident_response_plan` and `postmortem_record` evidence types |

**Citation resolution rate**: 100% (assuming LLM follows F5's instructions correctly)

**Evidence gaps to expect**:
- DQ: No artifacts matching `data_transform_test`
- IH: No artifacts matching `incident_response_plan` or `postmortem_record`

These gaps are intentional in the demo data — they demonstrate that the pipeline correctly identifies missing evidence.
