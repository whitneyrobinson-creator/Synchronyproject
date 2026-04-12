# F5 — SKILL.md: RCSA Control Narrative Generation — Data Model

**Feature Branch**: `f5-rcsa-skill`
**Created**: 2026-04-10
**Updated**: 2026-04-12
**Status**: Draft
**Phase**: 1

---

## Overview

This document defines the data structures that flow into, through, and out of the F5 SKILL.md. F5 is an orchestrator — it receives structured input from F6 scripts, reasons over it, and produces structured output. Every schema here represents a contract between F5 and F6.

**Key principle:** F5 never parses raw files. All input arrives pre-processed and structured by F6. F5's job is reasoning, not data processing.

**Control ID format:** Control IDs are uppercase 2–4 character strings (e.g., `AC`, `CM`, `DQ`, `IH`), enforced by F4's `control_library.schema.json` via regex `^[A-Z]{2,4}$`. F4's JSON files are the source of truth for control IDs.

---

## 1. Input Schemas

These are the structures F6 provides to the SKILL.md at the start of the workflow.

### 1.1 Control Library

The set of controls to assess. Framework-agnostic — any control set can be provided.

```yaml
# control_library.yaml
controls:
  - id: "AC"
    name: "Access Control"
    objective: "Ensure that access to systems and data is restricted to authorized users based on role and need."
    evidence_types:
      - "authentication_logic"
      - "authorization_checks"
      - "role_definitions"
      - "access_tests"

  - id: "CM"
    name: "Change Management"
    objective: "Ensure that changes to production systems follow a documented review and approval process."
    evidence_types:
      - "code_review_config"
      - "branch_protection"
      - "ci_cd_pipeline"
      - "approval_workflows"

  - id: "DQ"
    name: "Data Quality"
    objective: "Ensure that data inputs are validated, sanitized, and checked for integrity before processing."
    evidence_types:
      - "input_validation"
      - "schema_validation"
      - "data_integrity_checks"
      - "quality_tests"

  - id: "IH"
    name: "Incident Handling"
    objective: "Ensure that security incidents are detected, logged, and escalated through a defined response process."
    evidence_types:
      - "error_handling"
      - "logging_config"
      - "alerting_rules"
      - "incident_response_tests"
```

**Field definitions:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | Yes | Unique control identifier. Uppercase 2–4 characters (regex: `^[A-Z]{2,4}$`). Defined by F4's `control_library.schema.json`. |
| `name` | string | Yes | Human-readable control name |
| `objective` | string | Yes | What the control is supposed to achieve. Used by the LLM to assess whether evidence supports the objective. |
| `evidence_types` | list[string] | Yes | Types of artifacts that would constitute evidence for this control. Used by F6 for mapping and by F5 for confidence scoring. |

---

### 1.2 Artifact Registry

The complete inventory of artifacts found in the repository, built by F6's registry script.

```yaml
# artifact_registry.yaml
artifacts:
  - id: "ART-001"
    file_path: "src/auth/login.py"
    artifact_type: "authentication_logic"
    snippet: |
      def authenticate_user(username, password):
          """Validates credentials against the user store.
          Returns a session token on success, raises AuthError on failure."""
          user = user_store.get(username)
          if not user or not verify_hash(password, user.password_hash):
              raise AuthError("Invalid credentials")
          return generate_session_token(user)
    snippet_line_range: "12-19"
    file_size_bytes: 2048
    last_modified: "2026-03-15"

  - id: "ART-002"
    file_path: "src/auth/roles.py"
    artifact_type: "role_definitions"
    snippet: |
      ROLES = {
          "admin": ["read", "write", "delete", "manage_users"],
          "editor": ["read", "write"],
          "viewer": ["read"]
      }
    snippet_line_range: "1-6"
    file_size_bytes: 512
    last_modified: "2026-02-28"

  - id: "ART-003"
    file_path: "tests/test_auth.py"
    artifact_type: "access_tests"
    snippet: |
      def test_unauthorized_access_denied():
          """Verify that unauthenticated requests return 401."""
          response = client.get("/admin", headers={})
          assert response.status_code == 401
    snippet_line_range: "45-49"
    file_size_bytes: 3072
    last_modified: "2026-03-20"

  - id: "ART-004"
    file_path: ".github/workflows/ci.yml"
    artifact_type: "ci_cd_pipeline"
    snippet: |
      on:
        pull_request:
          branches: [main]
      jobs:
        test:
          runs-on: ubuntu-latest
          steps:
            - uses: actions/checkout@v4
            - run: pytest tests/
    snippet_line_range: "1-10"
    file_size_bytes: 1024
    last_modified: "2026-01-10"
```

**Field definitions:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | Yes | Unique artifact identifier. Format: `ART-[NUMBER]`. Assigned by F6. |
| `file_path` | string | Yes | Relative path from repository root |
| `artifact_type` | string | Yes | Classification of the artifact. Must match one of the `evidence_types` in the control library. |
| `snippet` | string | Yes | Relevant code/config excerpt. Extracted by F6 — not the full file. |
| `snippet_line_range` | string | Yes | Line range of the snippet within the file. Format: `"start-end"` |
| `file_size_bytes` | integer | No | Size of the full file. Informational only. |
| `last_modified` | string (date) | No | Last modification date. Informational only. |

**Constraints:**
- Snippet length is controlled by F6. The SKILL.md assumes snippets are short enough to fit in context.
- `artifact_type` must be a value that appears in at least one control's `evidence_types` list. Artifacts with unrecognized types are excluded by F6 before reaching F5.

---

### 1.3 Mapped Evidence

The output of F6's evidence mapping script. Links artifacts to controls based on `evidence_types` matching.

```yaml
# mapped_evidence.yaml
mappings:
  - control_id: "AC"
    control_name: "Access Control"
    mapped_artifacts:
      - artifact_id: "ART-001"
        artifact_type: "authentication_logic"
        relevance: "direct"
      - artifact_id: "ART-002"
        artifact_type: "role_definitions"
        relevance: "direct"
      - artifact_id: "ART-003"
        artifact_type: "access_tests"
        relevance: "direct"

  - control_id: "CM"
    control_name: "Change Management"
    mapped_artifacts:
      - artifact_id: "ART-004"
        artifact_type: "ci_cd_pipeline"
        relevance: "direct"

  - control_id: "DQ"
    control_name: "Data Quality"
    mapped_artifacts: []

  - control_id: "IH"
    control_name: "Incident Handling"
    mapped_artifacts: []
```

**Field definitions:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `control_id` | string | Yes | References `controls[].id` from the control library. Uppercase 2–4 characters. |
| `control_name` | string | Yes | Human-readable name (duplicated for LLM readability) |
| `mapped_artifacts` | list[object] | Yes | Artifacts mapped to this control. Empty list = GAP candidate. |
| `mapped_artifacts[].artifact_id` | string | Yes | References `artifacts[].id` from the artifact registry |
| `mapped_artifacts[].artifact_type` | string | Yes | The evidence type this artifact satisfies |
| `mapped_artifacts[].relevance` | string | Yes | `"direct"` or `"indirect"`. Direct = artifact type matches control's evidence type exactly. Indirect = partial or inferred match. |

**Key behaviors:**
- An artifact can appear in multiple controls' `mapped_artifacts` (multi-control artifact)
- An empty `mapped_artifacts` list triggers a GAP flag in the SKILL.md
- The `relevance` field feeds into the confidence rubric: `"direct"` counts as a strong signal, `"indirect"` counts as a weak signal

---

## 2. Output Schemas

These are the structures the SKILL.md produces after reasoning.

### 2.1 Primary Output: `rcsa_control_narratives.md`

Markdown document with YAML front matter. This is the auditor-facing deliverable.

**YAML Front Matter:**

```yaml
---
title: "RCSA Control Narrative Assessment"
generated: "2026-04-10T14:30:00Z"
controls_assessed: 4
controls_passing: 1
controls_low_confidence: 1
controls_gap: 2
framework: "Custom Demo Controls"
---
```

| Field | Type | Description |
|-------|------|-------------|
| `title` | string | Static title |
| `generated` | string (ISO 8601) | Timestamp of generation |
| `controls_assessed` | integer | Total number of controls in the input |
| `controls_passing` | integer | Controls with HIGH or MEDIUM confidence |
| `controls_low_confidence` | integer | Controls with LOW confidence |
| `controls_gap` | integer | Controls with zero evidence |
| `framework` | string | Name/description of the control framework used |

**Markdown Body Structure:**

```markdown
## Summary

| Control ID | Control Name | Confidence | Artifacts | Status |
|------------|-------------|------------|-----------|--------|
| AC | Access Control | HIGH | 3 | ✅ Assessed |
| CM | Change Management | LOW | 1 | ⚠️ Weak Evidence |
| DQ | Data Quality | GAP | 0 | 🔴 No Evidence |
| IH | Incident Handling | GAP | 0 | 🔴 No Evidence |

---

## AC — Access Control

**Confidence**: HIGH
**Artifacts Cited**: 3
**Evidence Types Covered**: authentication_logic, role_definitions, access_tests

The repository demonstrates a structured approach to access control.
Authentication is handled through a dedicated login module that validates
credentials against a user store and issues session tokens on success
[src/auth/login.py — credential validation and session token generation].
Role-based access is enforced through an explicit role definitions file that
maps roles to permitted operations [src/auth/roles.py — ROLES dictionary
with admin/editor/viewer permissions]. Automated tests verify that
unauthenticated requests are rejected [tests/test_auth.py —
test_unauthorized_access_denied asserts 401 response].

---

## DQ — Data Quality

**Confidence**: GAP
**Artifacts Cited**: 0

[GAP] No artifacts were mapped to the Data Quality control. No evidence of
input validation, schema validation, data integrity checks, or quality tests
was found in the repository. This control requires human review to determine
whether evidence exists outside the scanned artifacts or whether remediation
is needed.
```

**Per-control section structure:**

| Element | Required | Description |
|---------|----------|-------------|
| Heading | Yes | `## [control_id] — [control_name]` |
| Confidence | Yes | `HIGH`, `MEDIUM`, `LOW`, or `GAP` |
| Artifacts Cited | Yes | Count of artifacts referenced in the narrative |
| Evidence Types Covered | Yes | Which of the control's expected evidence types were found |
| Narrative | Yes | 3–5 sentences. Inline citations in format `[file_path — description]` |
| GAP flag | If applicable | `[GAP]` prefix when zero artifacts are mapped |

**Citation format:**
- In narratives: `[file_path — brief description of what the artifact shows]`
- Example: `[src/auth/login.py — credential validation and session token generation]`
- Citations must reference only artifacts present in the artifact registry
- Each citation must correspond to a specific `artifact_id` (validated by F6 post-generation)

---

### 2.2 Secondary Output: `validation_report.md`

Technical report for QA — verifies citation integrity and provides generation metadata.

```markdown
# Validation Report

**Generated**: 2026-04-10T14:30:00Z
**SKILL.md Version**: 1.0
**Controls Assessed**: 4

## Citation Resolution

| Citation | Artifact ID | File Path | Resolved | Match Type |
|----------|-------------|-----------|----------|------------|
| src/auth/login.py — credential validation... | ART-001 | src/auth/login.py | ✅ Yes | Exact |
| src/auth/roles.py — ROLES dictionary... | ART-002 | src/auth/roles.py | ✅ Yes | Exact |
| tests/test_auth.py — test_unauthorized... | ART-003 | tests/test_auth.py | ✅ Yes | Exact |
| .github/workflows/ci.yml — pytest execution... | ART-004 | .github/workflows/ci.yml | ✅ Yes | Exact |

**Total Citations**: 4
**Resolved**: 4
**Unresolved**: 0
**Resolution Rate**: 100%

## Confidence Scoring Audit

| Control ID | Confidence | Direct Artifacts | Indirect Artifacts | Rubric Match |
|------------|------------|-----------------|-------------------|--------------|
| AC | HIGH | 3 | 0 | ✅ ≥2 direct → HIGH |
| CM | LOW | 1 | 0 | ✅ 1 direct, 0 indirect → LOW |
| DQ | GAP | 0 | 0 | ✅ 0 artifacts → GAP |
| IH | GAP | 0 | 0 | ✅ 0 artifacts → GAP |

## Excluded Evidence

| Artifact ID | Control ID | Reason Excluded |
|-------------|------------|-----------------|
| (none in this run) | — | — |

## Warnings

- CM: Only 1 of 4 expected evidence types covered (ci_cd_pipeline). Consider reviewing for missing artifacts.
```

**Sections:**

| Section | Purpose |
|---------|---------|
| Citation Resolution | Verifies every citation in the narratives resolves to a real artifact in the registry |
| Confidence Scoring Audit | Shows the rubric calculation for each control's confidence tier |
| Excluded Evidence | Lists any artifacts that were excluded due to ambiguity, with reasons |
| Warnings | Flags controls with low evidence type coverage or other concerns |

---

## 3. Internal Data Flow

How data moves through the 6-step orchestration sequence defined in the SKILL.md.

```
Step 1: F6 — Validate Inputs
┌─────────────────────────────────────────────┐
│ IN:  control_library.yaml                   │
│      raw repository file listing            │
│ OUT: validation_result (pass/fail + errors) │
└─────────────────────────────────────────────┘
                    │
                    ▼
Step 2: F6 — Build Artifact Registry
┌─────────────────────────────────────────────┐
│ IN:  raw repository files                   │
│      control_library.yaml (evidence_types)  │
│ OUT: artifact_registry.yaml                 │
└─────────────────────────────────────────────┘
                    │
                    ▼
Step 3: F6 — Map Evidence to Controls
┌─────────────────────────────────────────────┐
│ IN:  artifact_registry.yaml                 │
│      control_library.yaml                   │
│ OUT: mapped_evidence.yaml                   │
└─────────────────────────────────────────────┘
                    │
                    ▼
Step 4: LLM — Generate Narratives
┌─────────────────────────────────────────────┐
│ IN:  control_library.yaml                   │
│      artifact_registry.yaml (for snippets)  │
│      mapped_evidence.yaml                   │
│ OUT: draft narratives + confidence + gaps   │
└─────────────────────────────────────────────┘
                    │
                    ▼
Step 5: F6 — Validate Citations
┌─────────────────────────────────────────────┐
│ IN:  draft narratives (from Step 4)         │
│      artifact_registry.yaml                 │
│ OUT: validation_report.md                   │
│      list of unresolved citations (if any)  │
└─────────────────────────────────────────────┘
                    │
                    ▼
Step 6: LLM — Assemble Final Output
┌─────────────────────────────────────────────┐
│ IN:  draft narratives (from Step 4)         │
│      validation results (from Step 5)       │
│ OUT: rcsa_control_narratives.md (final)     │
│      validation_report.md (final)           │
└─────────────────────────────────────────────┘
```

**Notes on the flow:**
- Steps 1–3 are deterministic (F6 scripts). The LLM calls these scripts and waits for results.
- Step 4 is where the LLM does its core reasoning work.
- Step 5 is deterministic (F6 script). The LLM calls this to verify its own citations.
- Step 6 is LLM reasoning — assembling the summary table, YAML front matter, and final document structure.
- If Step 5 finds unresolved citations, the LLM must fix them before producing final output in Step 6.

---

## 4. Schema Versioning

All schemas use a `v1` implicit version for the demo. Post-demo, if schemas evolve:

- Add an explicit `schema_version` field to the YAML front matter of each file
- F6 scripts validate schema version before processing
- The SKILL.md specifies which schema versions it supports

For the demo, version is implicit and all components assume `v1`.

---

*This document defines the data contracts for F5. All input schemas are produced by F6 scripts. All output schemas are produced by the SKILL.md. Changes to these schemas require updates to both F5 and F6.*
