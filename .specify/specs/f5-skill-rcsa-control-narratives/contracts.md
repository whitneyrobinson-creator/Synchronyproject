# F5 — SKILL.md: RCSA Control Narrative Generation — Contracts

**Feature Branch**: `f5-rcsa-skill`
**Created**: 2026-04-10
**Status**: Draft
**Phase**: 1

---

## Overview

This document defines the interface contracts between F5 (the SKILL.md orchestrator) and its two neighbors:

- **F6 (Scripts)** — the deterministic Python scripts that F5 calls as tools
- **F4 (Templates)** — the feature that will formalize F5's output structure into reusable templates

**Direction of data flow:**

```
F6 → F5:  Structured input (control library, artifact registry, mapped evidence)
F5 → F6:  Draft narratives for citation validation
F6 → F5:  Validation results (resolved/unresolved citations)
F5 → F4:  Output structure definition (F4 inherits and formalizes)
```

---

# Contract 1: F5 ↔ F6 (Scripts)

F5 calls F6 scripts as tools during the workflow and receives structured data back. F6 also validates F5's output after generation.

---

## 1.1 F6 Provides to F5

### Input Validation Result (Step 1)

**Script**: `validate_inputs`
**Called by**: SKILL.md (Step 1 of orchestration)

**Input to script:**
- `control_library.yaml`
- Raw repository file listing

**Output from script:**

```yaml
validation:
  status: "pass"  # or "fail"
  errors: []       # empty if pass
  # Example errors:
  # - "control_library.yaml: missing required field 'objective' in control CM-001"
  # - "control_library.yaml: duplicate control id 'AC-001'"
```

**Contract rules:**
- If `status` is `"fail"`, the SKILL.md MUST stop and report the errors. It MUST NOT attempt to reason over invalid data.
- If `status` is `"pass"`, the SKILL.md proceeds to Step 2.
- F6 validates: required fields present, no duplicate IDs, evidence_types is non-empty for each control.

---

### Artifact Registry (Step 2)

**Script**: `build_registry`
**Called by**: SKILL.md (Step 2 of orchestration)

**Input to script:**
- Raw repository files
- `control_library.yaml` (for evidence_types reference)

**Output from script:**

```yaml
artifacts:
  - id: "ART-001"          # string, required, unique, format: ART-[NUMBER]
    file_path: "src/..."    # string, required, relative from repo root
    artifact_type: "..."    # string, required, must match an evidence_type
    snippet: |              # string, required, relevant excerpt
      ...
    snippet_line_range: "12-19"  # string, required, format: "start-end"
    file_size_bytes: 2048        # integer, optional
    last_modified: "2026-03-15"  # string (date), optional
```

**Contract rules:**
- Every `artifact_type` MUST match at least one `evidence_types` value from the control library. F6 excludes artifacts with unrecognized types before passing to F5.
- Snippets MUST be short enough to fit within the LLM's context window when all snippets are combined. Snippet length is F6's responsibility.
- Artifact IDs MUST be unique and sequential.
- If no artifacts are found, F6 returns an empty list: `artifacts: []`

---

### Mapped Evidence (Step 3)

**Script**: `map_evidence`
**Called by**: SKILL.md (Step 3 of orchestration)

**Input to script:**
- `artifact_registry.yaml`
- `control_library.yaml`

**Output from script:**

```yaml
mappings:
  - control_id: "AC-001"       # string, required, references control library
    control_name: "Access Control"  # string, required
    mapped_artifacts:
      - artifact_id: "ART-001"     # string, required, references artifact registry
        artifact_type: "..."        # string, required
        relevance: "direct"         # string, required, "direct" or "indirect"
```

**Contract rules:**
- Every control in the control library MUST appear in the mappings, even if `mapped_artifacts` is empty.
- `relevance` values: `"direct"` = artifact_type exactly matches a control's evidence_type. `"indirect"` = partial or inferred match.
- A single artifact MAY appear in multiple controls' `mapped_artifacts`.
- F6 handles the mapping logic. F5 trusts the mapping but uses snippets as ground truth for reasoning.

---

## 1.2 F5 Provides to F6

### Draft Narratives for Validation (Step 4 → Step 5)

After the LLM generates narratives in Step 4, the draft output is passed to F6 for citation validation in Step 5.

**What F5 provides:**
- The complete draft of `rcsa_control_narratives.md` including all inline citations

**What F6 needs to extract:**
- Every citation in the format `[file_path — description]`
- The `file_path` portion of each citation

**Contract rules:**
- F5 MUST use the citation format `[file_path — description]` consistently so F6 can parse citations programmatically.
- F5 MUST only cite artifacts present in the artifact registry. (F6 validates this, but F5 is instructed to comply.)
- F5 MUST NOT modify artifact file paths — they must match exactly as provided in the registry.

---

## 1.3 F6 Returns to F5

### Citation Validation Results (Step 5 → Step 6)

**Script**: `validate_citations`
**Called by**: SKILL.md (Step 5 of orchestration)

**Input to script:**
- Draft narratives (from Step 4)
- `artifact_registry.yaml`

**Output from script:**

```yaml
citation_validation:
  total_citations: 4
  resolved: 4
  unresolved: 0
  resolution_rate: 1.0
  citations:
    - citation_text: "src/auth/login.py — credential validation..."
      file_path: "src/auth/login.py"
      artifact_id: "ART-001"
      resolved: true
      match_type: "exact"
  unresolved_list: []
```

**Contract rules:**
- If `unresolved > 0`, the SKILL.md MUST fix the broken citations before producing final output in Step 6. Options: remove the citation, replace with a valid one, or flag the issue.
- F6 matches citations by `file_path` against the artifact registry's `file_path` field. Match is exact string comparison.
- F6 does NOT evaluate whether the citation description is accurate — only whether the file path resolves.

---

## 1.4 F5 ↔ F6 Error Handling

| Scenario | F6 Responsibility | F5 Responsibility |
|----------|-------------------|-------------------|
| Invalid input files | Return `status: "fail"` with error list | Stop and report errors. Do not proceed. |
| No artifacts found | Return empty `artifacts: []` | Flag all controls as GAP |
| No mappings for a control | Return empty `mapped_artifacts: []` | Flag that control as GAP |
| Unresolved citations | Return `unresolved_list` with details | Fix or remove citations before final output |
| Script timeout | Return timeout error | Report the error. Do not retry automatically. |
| Malformed script output | N/A — F6 bug | Report that script output was unparseable. Do not guess. |

---

# Contract 2: F5 ↔ F4 (Templates)

F4 inherits F5's output structure and formalizes it into reusable, parameterized templates. This contract is lighter — F4 consumes what F5 defines, not the other way around.

---

## 2.1 What F5 Defines (F4 Inherits)

F5 owns the structure of both output documents. F4's job is to take these structures and make them reusable across different control frameworks and repositories.

### Output 1: `rcsa_control_narratives.md`

F4 inherits:

| Element | What F4 Formalizes |
|---------|-------------------|
| YAML front matter fields | Template variables (e.g., `{{title}}`, `{{controls_assessed}}`) |
| Summary table columns | Fixed column structure with dynamic row generation |
| Per-control section layout | Repeatable template block per control |
| Citation format | `[file_path — description]` as a standard pattern |
| GAP flag format | `[GAP]` prefix as a standard pattern |
| Confidence tiers | `HIGH`, `MEDIUM`, `LOW`, `GAP` as an enum |

### Output 2: `validation_report.md`

F4 inherits:

| Element | What F4 Formalizes |
|---------|-------------------|
| Citation resolution table | Fixed column structure with dynamic row generation |
| Confidence scoring audit table | Fixed column structure with rubric reference |
| Excluded evidence section | Optional section, included only when exclusions exist |
| Warnings section | Optional section, included only when warnings exist |

---

## 2.2 Contract Rules

| Rule | Description |
|------|-------------|
| **F5 owns output structure** | F5 defines what the output looks like. F4 cannot change the structure — only parameterize it. |
| **F4 owns template syntax** | F4 decides how to express templates (Jinja2, Mustache, etc.). F5 doesn't care about template syntax. |
| **Changes flow F5 → F4** | If F5 changes its output structure, F4 must update its templates. Not the reverse. |
| **F4 must preserve all fields** | F4 templates must include every field F5 defines. F4 cannot drop fields for simplicity. |
| **F4 can add presentation** | F4 can add styling, formatting, or layout improvements as long as all F5 data is preserved. |

---

## 2.3 Handoff Checklist

Before F4 begins template work, F5 must provide:

- [ ] Final `rcsa_control_narratives.md` structure (from data-model.md)
- [ ] Final `validation_report.md` structure (from data-model.md)
- [ ] List of all dynamic fields (anything that changes per run)
- [ ] List of all static fields (anything that stays the same across runs)
- [ ] At least one complete example output (from quickstart.md test run)
- [ ] Confidence tier rubric (so F4 can document it in the template)

---

## 2.4 F5 ↔ F4 Error Handling

| Scenario | Responsibility |
|----------|---------------|
| F5 output doesn't match documented structure | F5 bug — fix the SKILL.md |
| F4 template drops a field | F4 bug — fix the template |
| New control framework needs different fields | F5 decides whether to add fields. F4 updates templates after. |
| Output format changes (e.g., markdown → HTML) | Joint decision. F5 updates output instructions, F4 updates templates. |

---

# Versioning

All schemas and contracts are implicitly `v1` for the demo. Post-demo:

- Add `schema_version: "1.0"` to the root of each YAML output
- F5 specifies which schema versions it supports in the SKILL.md header
- F6 validates schema version before processing
- F4 templates specify which F5 output version they support

---

*This document is owned jointly by F5, F6, and F4. Changes to any contract require review by the owners of both sides of the interface.*
