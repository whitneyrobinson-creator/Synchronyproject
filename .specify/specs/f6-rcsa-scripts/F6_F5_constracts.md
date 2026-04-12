# Contract: F6 ↔ F5 (Scripts ↔ SKILL.md)

**Feature**: F6 — RCSA Scripts | **Date**: 2026-04-12 | **Status**: Draft
**Parties**: F6 (scripts) ↔ F5 (agent orchestration)

---

## Purpose

This contract defines the interface between F6's Python scripts and F5's SKILL.md. F5 orchestrates the pipeline — it decides when to call each script, what data to pass, and when to invoke the LLM. F6 provides the scripts that F5 calls. Both sides must agree on data shapes, invocation order, and degradation behavior.

---

## 1. Invocation Order

F5's SKILL.md calls F6 scripts in this exact sequence:

| Step | Script | What F5 Passes In | What F6 Returns |
|---|---|---|---|
| 1 | `validate_input.py` | Paths to 3 JSON files | `ValidationResult` |
| 2 | `build_registry.py` | Paths to 3 JSON files | `ArtifactRegistry` |
| 3 | `map_evidence.py` | `ArtifactRegistry` + control library data | `MappedEvidence` |
| 4 | `orchestrate_llm.py` | `MappedEvidence` | Formatted LLM prompt (pre-LLM) |
| — | **F5 invokes LLM** | Formatted prompt from step 4 | Raw LLM response (Markdown) |
| 4b | `orchestrate_llm.py` | Raw LLM response + `MappedEvidence` | `LLMResponse` (parsed) |
| 5 | `validate_citations.py` | `LLMResponse` + `ArtifactRegistry` | `CitationValidationResult` |
| 6 | `write_output.py` | `LLMResponse` + `CitationValidationResult` + `MappedEvidence` | Output file paths |

**Key detail — Step 4 is split:**
- Step 4 (pre-LLM): `orchestrate_llm.py` formats the mapped evidence into a prompt
- F5 takes that prompt and sends it to the LLM (Claude Sonnet via Cursor agent runtime)
- Step 4b (post-LLM): `orchestrate_llm.py` parses the raw LLM response into structured `LLMResponse`

F6 never calls the LLM directly. F5 owns LLM invocation.

---

## 2. Data Shapes

All data passed between F5 and F6 scripts is JSON. Full schemas are defined in `data-model.md`. This section summarizes the contract-critical fields.

### 2.1 ValidationResult (Step 1 → F5)

```json
{
  "status": "pass" | "fail",
  "files_validated": [...]
}
```

**F5's contract obligation:**
- If `status` is `"fail"` → F5 MUST stop the pipeline and report errors to the user
- If `status` is `"pass"` → F5 proceeds to Step 2

---

### 2.2 ArtifactRegistry (Step 2 → Steps 3, 5)

```json
{
  "registry": {
    "artifacts": { "ART-001": {...}, ... },
    "tests": { "TST-001": {...}, ... },
    "file_path_index": { "auth/oauth_config.yaml": "ART-001", ... },
    "stats": { "total_artifacts": 15, "total_tests": 8, "total_entries": 23 }
  }
}
```

**F5's contract obligation:**
- F5 MUST pass this to `map_evidence.py` (Step 3) and `validate_citations.py` (Step 5)
- F5 MUST hold this in memory — it is used twice

---

### 2.3 MappedEvidence (Step 3 → Steps 4, 4b, 6)

```json
{
  "mappings": [
    {
      "control_id": "AC",
      "control_name": "...",
      "control_objective": "...",
      "mapped_artifacts": [...],
      "mapped_tests": [...],
      "evidence_type_coverage": {...},
      "gap_flag": false,
      "artifact_count": 5,
      "test_count": 3
    }
  ],
  "summary": {...}
}
```

**F5's contract obligation:**
- F5 MUST pass this to `orchestrate_llm.py` (Step 4)
- F5 MUST also pass this to `write_output.py` (Step 6) for evidence coverage reporting
- F5 MUST hold this in memory — it is used three times (Steps 4, 4b, 6)

---

### 2.4 Formatted LLM Prompt (Step 4 → F5)

`orchestrate_llm.py` returns a string (not JSON) — the formatted prompt for the LLM.

**F5's contract obligation:**
- F5 takes this string and sends it to the LLM as the user message / prompt
- F5 MUST NOT modify the prompt content
- F5 MAY wrap it in additional system instructions per the SKILL.md

---

### 2.5 Raw LLM Response (F5 → Step 4b)

F5 passes the raw LLM output (Markdown string) back to `orchestrate_llm.py` for parsing.

**F6's contract obligation:**
- `orchestrate_llm.py` MUST handle these cases:
  - **Valid response**: Markdown with control sections and citations → parse into `LLMResponse`
  - **Empty response**: LLM returned nothing → return degraded `LLMResponse`
  - **Malformed response**: LLM returned something but it doesn't match expected structure → attempt partial parse, degrade controls that failed
  - **Timeout / error**: F5 signals the LLM failed → return degraded `LLMResponse`

**F5's contract obligation:**
- If the LLM fails (timeout, error, empty), F5 MUST still call `orchestrate_llm.py` with an indicator of failure (e.g., empty string or error flag) rather than skipping the script
- This ensures the pipeline always completes, even in degraded mode

---

### 2.6 LLMResponse (Step 4b → Steps 5, 6)

```json
{
  "status": "success" | "degraded",
  "narratives": {
    "AC": {
      "confidence": "HIGH",
      "narrative_text": "...",
      "artifacts_cited": ["ART-001", ...],
      "tests_cited": ["TST-001", ...],
      "gap_flag": false
    }
  },
  "yaml_front_matter": {...}
}
```

**F5's contract obligation:**
- F5 MUST pass this to `validate_citations.py` (Step 5) and `write_output.py` (Step 6)
- If `status` is `"degraded"`, F5 still proceeds — the pipeline produces raw evidence output instead of narratives

---

### 2.7 CitationValidationResult (Step 5 → Step 6)

```json
{
  "total_citations": 15,
  "resolved": 15,
  "unresolved": 0,
  "resolution_rate": 1.0,
  "citations": [...],
  "unresolved_list": [...]
}
```

**F5's contract obligation:**
- F5 MUST pass this to `write_output.py` (Step 6)
- F5 does NOT stop the pipeline if citations are unresolved — unresolved citations are reported in the validation report, not treated as errors

---

## 3. LLM Response Structure

F5's SKILL.md instructs the LLM to produce output in this structure. F6 parses it.

### Expected LLM Output Format

```markdown
## AC — Access Control

**Confidence**: HIGH

[Narrative text with inline citations in [file_path — description] format]

---

## CM — Change Management

**Confidence**: MEDIUM

[Narrative text with inline citations]

---

[Repeats for each control]
```

**Parsing contract:**
- Each control section starts with `## [CONTROL_ID] — [Control Name]`
- Confidence tier appears on the line starting with `**Confidence**:`
- Valid confidence values: `HIGH`, `MEDIUM`, `LOW`, `GAP`
- Sections are separated by `---`
- Citations within narrative text follow `[file_path — description]` format

**If the LLM deviates:**
- Missing section for a control → that control is marked as degraded, others are unaffected
- Missing confidence line → default to `"UNAVAILABLE"`
- Malformed citations → caught by `validate_citations.py` in Step 5
- Extra content outside control sections → ignored

---

## 4. Confidence Tier Rubric

Defined by F5, consumed by F6. F6 does not calculate tiers — it extracts what the LLM assigns.

| Tier | Meaning | Assigned When |
|---|---|---|
| `HIGH` | Strong evidence across all evidence types | Multiple artifacts + tests covering all evidence types |
| `MEDIUM` | Partial evidence — some types covered | Some evidence types have artifacts, others don't |
| `LOW` | Weak evidence — minimal coverage | Few artifacts, few or no tests |
| `GAP` | No evidence found | Zero artifacts AND zero tests mapped |
| `UNAVAILABLE` | LLM did not assign a tier | F6 default when tier is missing or LLM is unavailable |

---

## 5. Graceful Degradation Protocol

This is the most critical part of the contract. It defines what happens when things go wrong.

### Degradation Scenarios

| Scenario | Who Detects | What Happens | Pipeline Continues? |
|---|---|---|---|
| Input validation fails | F6 (`validate_input.py`) | Returns `status: "fail"` with errors | ❌ No — F5 stops |
| LLM times out | F5 | F5 passes empty/error to `orchestrate_llm.py` | ✅ Yes — degraded mode |
| LLM returns empty | F5 | F5 passes empty string to `orchestrate_llm.py` | ✅ Yes — degraded mode |
| LLM returns malformed output | F6 (`orchestrate_llm.py`) | Parses what it can, degrades the rest | ✅ Yes — partial degradation |
| LLM hallucinates citations | F6 (`validate_citations.py`) | Flags unresolved citations in report | ✅ Yes — reported in output |
| One control section missing from LLM | F6 (`orchestrate_llm.py`) | That control marked degraded, others normal | ✅ Yes — per-control degradation |

### Degraded Output

When the pipeline runs in degraded mode, `write_output.py` produces:

- `rcsa_control_narratives.md` with:
  - YAML front matter including `mode: "degraded"`
  - Summary table showing `UNAVAILABLE` for degraded controls
  - Raw evidence listing instead of LLM narratives for degraded controls
  - Normal narratives for any controls that succeeded

- `validation_report.md` with:
  - Note that the pipeline ran in degraded mode
  - Evidence coverage table (still populated from `MappedEvidence`)
  - Citation table empty or partial for degraded controls

**The pipeline ALWAYS produces output files.** The only exception is input validation failure (Step 1).

---

## 6. Error Signaling

### F6 → F5 (Script errors)

| Signal | Meaning | F5 Action |
|---|---|---|
| Exit code 0 + JSON with `status: "pass"` or `status: "success"` | Script succeeded | Proceed to next step |
| Exit code 0 + JSON with `status: "fail"` | Validation failed (Step 1 only) | Stop pipeline, report errors |
| Exit code 0 + JSON with `status: "degraded"` | LLM failed but pipeline can continue | Proceed in degraded mode |
| Exit code 1 | Unexpected script error (bug) | Stop pipeline, report error |

### F5 → F6 (LLM errors)

| Signal | Meaning | F6 Action |
|---|---|---|
| Valid Markdown string | LLM succeeded | Parse normally |
| Empty string `""` | LLM returned nothing | Return degraded `LLMResponse` |
| Error flag / null | LLM timed out or errored | Return degraded `LLMResponse` |

---

## 7. Performance Budget

| Owner | Component | Budget |
|---|---|---|
| F6 | All deterministic scripts (Steps 1-3, 5-6) | < 30 seconds total |
| F5 | LLM invocation (per control) | ≤ 30 seconds |
| F5 | LLM invocation (all controls) | ≤ 120 seconds |
| F6 | `orchestrate_llm.py` formatting + parsing | < 5 seconds |
| **Total** | **Full pipeline** | **< 155 seconds worst case** |

F6's 60-second target (from plan.md) covers only the deterministic portions. LLM thinking time is additive and governed by F5's budget.

---

## 8. Breaking Changes

### Changes in F5 that break F6:

| Change | Impact on F6 |
|---|---|
| Change invocation order | Scripts may receive wrong input data |
| Change LLM response structure (headings, delimiters) | `orchestrate_llm.py` parser breaks |
| Change confidence tier values | `orchestrate_llm.py` tier extraction fails |
| Change citation format instructions to LLM | `validate_citations.py` regex fails |
| Stop passing `ArtifactRegistry` to Step 5 | `validate_citations.py` cannot resolve citations |
| Skip calling `orchestrate_llm.py` on LLM failure | Pipeline cannot degrade gracefully |

### Changes in F6 that break F5:

| Change | Impact on F5 |
|---|---|
| Change return data shape of any script | F5 passes wrong data to next script |
| Change exit code conventions | F5 misinterprets success/failure |
| Add new required input parameters | F5 doesn't know to pass them |
| Remove a script | F5's step sequence breaks |

---

*This contract is owned jointly by F5 and F6. Changes to either side require review by both. The SKILL.md (F5) and the script interfaces (F6) must stay in sync.*
