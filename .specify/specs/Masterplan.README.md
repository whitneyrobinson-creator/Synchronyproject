# RCSA Control Narrative Skill — Master Reference

> **Project**: Synchrony Documentation Automation Skillset  
> **Skill**: RCSA Control Narrative Generation (Skill 2)  
> **Features**: F4 (Assets), F5 (SKILL.md), F6 (Scripts)  
> **Last Updated**: 2026-04-12  
> **Demo Day Deadline**: May 7, 2026

This document is the home base for the RCSA Control Narrative skill. It tells you what the skill does, how the three features work together, and what every file and data structure in the pipeline is. For deep technical detail on any specific feature, go to that feature's plan.

---

# Table of Contents

- [What This Skill Does](#what-this-skill-does)
- [The Three Features](#the-three-features)
- [Repository Layout](#repository-layout)
- [End-to-End Workflow](#end-to-end-workflow)
  - [Step 1 — User Provides Three JSON Files](#step-1--user-provides-three-json-files)
  - [Step 2 — Validate Input](#step-2--validate-input)
  - [Step 3 — Build Artifact Registry](#step-3--build-artifact-registry)
  - [Step 4 — Map Evidence to Controls](#step-4--map-evidence-to-controls)
  - [Step 5 — LLM Generates Narratives](#step-5--llm-generates-narratives)
  - [Step 6 — Validate Citations](#step-6--validate-citations)
  - [Step 7 — Assemble Final Output](#step-7--assemble-final-output)
- [Every File in the Pipeline](#every-file-in-the-pipeline)
- [Key Rules That Govern the Whole Skill](#key-rules-that-govern-the-whole-skill)
  - [Confidence Tiers](#confidence-tiers)
  - [Citation Format](#citation-format)
  - [Gap Handling](#gap-handling)
  - [Placeholder Values — Degraded Mode](#placeholder-values--degraded-mode)
  - [Template Authority](#template-authority)
  - [Graceful Degradation](#graceful-degradation)
  - [Error Signaling Between F5 and F6](#error-signaling-between-f5-and-f6)
  - [Performance Budget](#performance-budget)
  - [Sample Data Coverage Design](#sample-data-coverage-design)
  - [Demo-Scale Constraints](#demo-scale-constraints)
- [Cross-Feature Dependencies](#cross-feature-dependencies)
- [Feature Ownership Summary](#feature-ownership-summary)

---

# What This Skill Does

A user provides three JSON files — an **artifact index** listing repository files, a **test catalog** listing test files, and a **control library** defining compliance controls. The skill maps evidence to controls and produces two documents:

1. **`rcsa_control_narratives.md`** — Citation-backed narratives for each compliance control (3–5 sentences each), with a summary table showing confidence tiers and gap flags, plus inline citations in `[file_path — description]` format.

2. **`validation_report.md`** — A quality report showing citation resolution statistics, evidence coverage per control, missing evidence types, and items flagged for human review.

The goal is to take a task that currently requires a compliance analyst to manually trace repository artifacts to controls — a process that takes hours and is error-prone — and reduce it to minutes, while producing output that is more consistent, more traceable, and more audit-ready than a human-written first draft.

**The system MUST never imply compliance without proof.** If no evidence exists for a control, it flags a GAP — it does not generate a compliance narrative.

---

# The Three Features

The skill is built from three features. Each owns a distinct layer.

| Feature | Name | What It Owns | Simple Version |
| ------- | ---- | ------------ | -------------- |
| **F4** | Assets — RCSA | The static files — output templates, sample input data, citation format specification, and control definitions | The blank forms + sample data |
| **F5** | SKILL.md — RCSA | The LLM reasoning layer — orchestrates the 6-step workflow, generates narratives with inline citations, flags gaps, assigns confidence tiers | The LLM thinks |
| **F6** | Scripts — RCSA | The deterministic pipeline — validates input, builds the artifact registry, maps evidence, validates citations, populates templates, writes output | The code works |

**The rule of thumb:** If it requires judgment, language, or orchestration decisions, F5 owns it. If it's deterministic logic, F6 owns it. If it's a static file the pipeline reads, F4 owns it.

## Build Order

Unlike Skill 1 (which built SKILL.md first), Skill 2 builds Assets (F4) first because the RCSA skill's templates, sample data schemas, and citation format must be locked before F5 can reference them and F6 can validate against them.

| Order | Feature | Rationale |
| ----- | ------- | --------- |
| 1 | F4 — Assets: RCSA | Establish the static foundation — templates, sample data, control library, and citation format |
| 2 | F5 — SKILL.md: RCSA | Define the LLM orchestration instructions that reference F4 assets and call F6 scripts |
| 3 | F6 — Scripts: RCSA | Build the deterministic pipeline that validates inputs, supports F5, and produces final outputs |

---

# Repository Layout

All files for this skill live under `skills/rcsa/`. Nothing in `specs/` runs at runtime — it's planning documentation only.

```text
synchrony-doc-automation/
│
├── .gitignore
│
├── skills/
│   └── rcsa/
│       ├── SKILL.md                             ← F5: LLM instruction file
│       │
│       ├── scripts/                             ← F6: 6 Python pipeline scripts
│       │   ├── validate_input.py
│       │   ├── build_registry.py
│       │   ├── map_evidence.py
│       │   ├── orchestrate_llm.py
│       │   ├── validate_citations.py
│       │   └── write_output.py
│       │
│       ├── scripts/tests/                       ← Dev only — NOT delivered to Synchrony
│       │   ├── test_validate_input.py
│       │   ├── test_build_registry.py
│       │   ├── test_map_evidence.py
│       │   ├── test_orchestrate_llm.py
│       │   ├── test_validate_citations.py
│       │   ├── test_write_output.py
│       │   └── mocks/
│       │       ├── mock_llm_response_success.md
│       │       ├── mock_llm_response_malformed.md
│       │       └── mock_llm_response_empty.md
│       │
│       ├── assets/                              ← F4: static asset package
│       │   ├── templates/
│       │   │   ├── rcsa_control_narratives_template.md
│       │   │   └── validation_report_template.md
│       │   ├── sample-data/
│       │   │   ├── sample_artifact_index.json
│       │   │   ├── sample_test_catalog.json
│       │   │   └── sample_control_library.json
│       │   └── citation_format.md
│       │
│       └── output/                              ← Created at runtime, NOT checked in
│           ├── rcsa_control_narratives.md
│           └── validation_report.md
│
└── specs/                                       ← Planning docs only, not runtime
    ├── f4-rcsa-assets/
    ├── 005-skill-rcsa/
    └── f6-rcsa-scripts/
```

**Notes:**

- `output/` is created at runtime by `write_output.py`. It is not checked into the repository.
- RCSA assets are co-located inside `skills/rcsa/assets/` because F4, F5, and F6 form a single agent skill.
- `scripts/tests/` contains mocked LLM responses for tier 2 testing. Not included in Synchrony handoff.
- RCSA scripts use Python 3.11 standard library only — zero third-party dependencies.
- For demo day, the LLM step is performed manually via Claude Desktop. No programmatic API calls are made.
- All `output/` directories must be in `.gitignore`.

---

# End-to-End Workflow

Here is exactly what happens from the moment a user provides their input files to the moment they receive their deliverables.

---

## Step 1 — User Provides Three JSON Files

The user provides three JSON files to Claude and asks it to generate RCSA control narratives.

### 1a. Control Library

`sample_control_library.json`

```json
{
  "_meta": { "scale": "demo", "note": "Replace with production data post-handoff" },
  "controls": [
    {
      "id": "AC",
      "name": "Access Control",
      "objective": "Ensure system access is restricted to authorized users...",
      "evidence_types": [
        { "id": "auth_config", "label": "Authentication Configuration", "description": "..." },
        { "id": "rbac_policy", "label": "RBAC Policy Definition", "description": "..." },
        { "id": "session_management", "label": "Session Management Configuration", "description": "..." }
      ]
    },
    {
      "id": "CM",
      "name": "Change Management",
      "objective": "Ensure all changes to production systems follow approved processes...",
      "evidence_types": [
        { "id": "change_approval_record", "label": "Change Approval Record", "description": "..." },
        { "id": "deployment_config", "label": "Deployment Configuration", "description": "..." },
        { "id": "rollback_procedure", "label": "Rollback Procedure", "description": "..." }
      ]
    },
    {
      "id": "DQ",
      "name": "Data Quality",
      "objective": "Ensure data inputs are validated and transformations preserve integrity",
      "evidence_types": [
        { "id": "input_validation_rule", "label": "Input Validation Rule", "description": "..." },
        { "id": "data_transform_test", "label": "Data Transformation Test", "description": "..." }
      ]
    },
    {
      "id": "IH",
      "name": "Incident Handling",
      "objective": "Ensure security incidents are detected, responded to, and reviewed",
      "evidence_types": [
        { "id": "monitoring_config", "label": "Monitoring Configuration", "description": "..." },
        { "id": "incident_response_plan", "label": "Incident Response Plan", "description": "..." },
        { "id": "postmortem_record", "label": "Postmortem Record", "description": "..." }
      ]
    }
  ]
}
```

### 1b. Artifact Index

`sample_artifact_index.json` — 15 artifacts across `auth/`, `deploy/`, `data/`, `monitoring/`.

```json
{
  "_meta": { "scale": "demo", "note": "Replace with production data post-handoff" },
  "artifacts": [
    { "id": "ART-001", "file_path": "auth/oauth_config.yaml", "file_type": "yaml", "description": "OAuth2 provider configuration..." },
    { "id": "ART-002", "file_path": "auth/mfa_settings.json", "file_type": "json", "description": "Multi-factor authentication settings..." },
    { "id": "ART-003", "file_path": "auth/rbac_roles.yaml", "file_type": "yaml", "description": "Role-based access control definitions..." }
  ]
}
```

### 1c. Test Catalog

`sample_test_catalog.json` — 8 tests. Distribution: AC (3), CM (3), DQ (1), IH (1).

```json
{
  "_meta": { "scale": "demo", "note": "Replace with production data post-handoff" },
  "tests": [
    { "id": "TST-001", "file_path": "tests/auth/test_rbac_permissions.py", "file_type": "python", "description": "Verifies RBAC role assignments...", "controls_relevant": ["AC"] }
  ]
}
```

---

## Step 2 — Validate Input

**Script:** `validate_input.py`

F6 checks that all three JSON files are valid before anything else runs. If anything is wrong, the pipeline stops here with a clear error message listing every problem.

### What It Checks

- Do all 3 JSON files exist at expected paths?
- Are they valid JSON?
- Does the control library have a `controls` array (non-empty)?
- Does every control have `id`, `name`, `objective`, and `evidence_types`?
- Does every control ID match the pattern `^[A-Z]{2,4}$`?
- Are there duplicate IDs in any file?
- Does every evidence type have `id`, `label`, and `description`?
- Does every artifact ID match `ART-NNN` pattern?
- Does every test ID match `TST-NNN` pattern?
- Does every value in `controls_relevant` reference a valid control ID?

### Output — ValidationResult

Passed in-memory to F5.

**Success:**

```json
{
  "status": "pass",
  "files_validated": [
    { "file": "control_library", "status": "pass", "errors": [] },
    { "file": "artifact_index", "status": "pass", "errors": [] },
    { "file": "test_catalog", "status": "pass", "errors": [] }
  ]
}
```

**Failure:**

```json
{
  "status": "fail",
  "files_validated": [
    {
      "file": "control_library",
      "status": "fail",
      "errors": [
        "Missing required field: controls[2].objective",
        "Invalid control ID 'access_control' — must match pattern ^[A-Z]{2,4}$"
      ]
    },
    { "file": "artifact_index", "status": "pass", "errors": [] },
    {
      "file": "test_catalog",
      "status": "fail",
      "errors": [
        "tests[3].controls_relevant contains 'XX' which is not a valid control ID"
      ]
    }
  ]
}
```

> **If `status` is `"fail"`, the pipeline stops. F5 reports the errors to the user. No downstream scripts execute.**

---

## Step 3 — Build Artifact Registry

**Script:** `build_registry.py`

F6 reads all three validated JSON files and builds a unified in-memory registry with O(1) lookups by ID and by file path. This registry is the single source of truth for citation validation later.

### Output — ArtifactRegistry

In-memory, not persisted to disk.

```json
{
  "registry": {
    "artifacts": {
      "ART-001": {
        "id": "ART-001",
        "file_path": "auth/oauth_config.yaml",
        "file_type": "yaml",
        "description": "OAuth2 provider configuration...",
        "source": "artifact_index"
      }
    },
    "tests": {
      "TST-001": {
        "id": "TST-001",
        "file_path": "tests/auth/test_rbac_permissions.py",
        "file_type": "python",
        "description": "Verifies RBAC role assignments...",
        "controls_relevant": ["AC"],
        "source": "test_catalog"
      }
    },
    "file_path_index": {
      "auth/oauth_config.yaml": "ART-001",
      "auth/mfa_settings.json": "ART-002",
      "auth/rbac_roles.yaml": "ART-003",
      "auth/rbac_permissions_matrix.json": "ART-004",
      "auth/session_config.yaml": "ART-005",
      "deploy/change_approval_workflow.yaml": "ART-006",
      "deploy/pr_review_policy.md": "ART-007",
      "deploy/ci_pipeline.yaml": "ART-008",
      "deploy/terraform_main.tf": "ART-009",
      "deploy/rollback_runbook.md": "ART-010",
      "deploy/rollback_script.sh": "ART-011",
      "data/input_schema.json": "ART-012",
      "data/validation_rules.yaml": "ART-013",
      "monitoring/alerting_rules.yaml": "ART-014",
      "monitoring/dashboard_config.json": "ART-015",
      "tests/auth/test_rbac_permissions.py": "TST-001",
      "tests/auth/test_session_timeout.py": "TST-002",
      "tests/auth/test_oauth_flow.py": "TST-003",
      "tests/deploy/test_approval_gate.py": "TST-004",
      "tests/deploy/test_rollback.sh": "TST-005",
      "tests/deploy/test_deployment_pipeline.py": "TST-006",
      "tests/data/test_input_validation.py": "TST-007",
      "tests/monitoring/test_alert_triggers.py": "TST-008"
    },
    "stats": {
      "total_artifacts": 15,
      "total_tests": 8,
      "total_entries": 23
    }
  }
}
```

### Why This Structure

- **`artifacts`** and **`tests`** are keyed by ID (not arrays) for fast lookup during evidence mapping.
- **`file_path_index`** is the critical structure for citation validation — when the LLM cites `[auth/oauth_config.yaml — ...]`, F6 looks up the file path here to confirm it exists.
- **`source`** field tracks whether an entry came from the artifact index or test catalog.

If no artifacts are found, F6 returns an empty registry. F5 flags all controls as GAP.

---

## Step 4 — Map Evidence to Controls

**Script:** `map_evidence.py`

F6 takes the artifact registry and control library and deterministically maps artifacts and tests to controls based on evidence type matching. Every control in the library appears in the output, even if it has no evidence.

### Mapping Logic

1. For each control, collect its `evidence_types[].id` values
2. For each artifact, determine which evidence type it matches based on its `description` and `file_path` context
3. For each test, read `controls_relevant[]` to directly assign it to controls
4. An artifact can map to multiple controls if it matches evidence types from more than one

### Output — MappedEvidence

In-memory. Used three times: Steps 5, 6, and 7.

```json
{
  "mappings": [
    {
      "control_id": "AC",
      "control_name": "Access Control",
      "control_objective": "Ensure system access is restricted to authorized users...",
      "mapped_artifacts": [
        { "id": "ART-001", "file_path": "auth/oauth_config.yaml", "evidence_type_matched": "auth_config" },
        { "id": "ART-002", "file_path": "auth/mfa_settings.json", "evidence_type_matched": "auth_config" },
        { "id": "ART-003", "file_path": "auth/rbac_roles.yaml", "evidence_type_matched": "rbac_policy" },
        { "id": "ART-004", "file_path": "auth/rbac_permissions_matrix.json", "evidence_type_matched": "rbac_policy" },
        { "id": "ART-005", "file_path": "auth/session_config.yaml", "evidence_type_matched": "session_management" }
      ],
      "mapped_tests": [
        { "id": "TST-001", "file_path": "tests/auth/test_rbac_permissions.py" },
        { "id": "TST-002", "file_path": "tests/auth/test_session_timeout.py" },
        { "id": "TST-003", "file_path": "tests/auth/test_oauth_flow.py" }
      ],
      "evidence_type_coverage": {
        "auth_config": ["ART-001", "ART-002"],
        "rbac_policy": ["ART-003", "ART-004"],
        "session_management": ["ART-005"]
      },
      "gap_flag": false,
      "artifact_count": 5,
      "test_count": 3
    },
    {
      "control_id": "CM",
      "control_name": "Change Management",
      "mapped_artifacts": ["ART-006 through ART-011 (6 artifacts)"],
      "mapped_tests": ["TST-004 through TST-006 (3 tests)"],
      "evidence_type_coverage": {
        "change_approval_record": ["ART-006", "ART-007"],
        "deployment_config": ["ART-008", "ART-009"],
        "rollback_procedure": ["ART-010", "ART-011"]
      },
      "gap_flag": false,
      "artifact_count": 6,
      "test_count": 3
    },
    {
      "control_id": "DQ",
      "control_name": "Data Quality",
      "mapped_artifacts": [
        { "id": "ART-012", "file_path": "data/input_schema.json", "evidence_type_matched": "input_validation_rule" },
        { "id": "ART-013", "file_path": "data/validation_rules.yaml", "evidence_type_matched": "input_validation_rule" }
      ],
      "mapped_tests": [
        { "id": "TST-007", "file_path": "tests/data/test_input_validation.py" }
      ],
      "evidence_type_coverage": {
        "input_validation_rule": ["ART-012", "ART-013"],
        "data_transform_test": []
      },
      "gap_flag": false,
      "artifact_count": 2,
      "test_count": 1
    },
    {
      "control_id": "IH",
      "control_name": "Incident Handling",
      "mapped_artifacts": [
        { "id": "ART-014", "file_path": "monitoring/alerting_rules.yaml", "evidence_type_matched": "monitoring_config" },
        { "id": "ART-015", "file_path": "monitoring/dashboard_config.json", "evidence_type_matched": "monitoring_config" }
      ],
      "mapped_tests": [
        { "id": "TST-008", "file_path": "tests/monitoring/test_alert_triggers.py" }
      ],
      "evidence_type_coverage": {
        "monitoring_config": ["ART-014", "ART-015"],
        "incident_response_plan": [],
        "postmortem_record": []
      },
      "gap_flag": false,
      "artifact_count": 2,
      "test_count": 1
    }
  ],
  "summary": {
    "total_controls": 4,
    "controls_with_evidence": 4,
    "controls_with_gaps": 0,
    "total_artifacts_mapped": 15,
    "total_tests_mapped": 8,
    "unmapped_artifacts": [],
    "unmapped_tests": []
  }
}
```

### Important Notes

- **`gap_flag`** is `true` ONLY when BOTH `mapped_artifacts` AND `mapped_tests` are empty. DQ and IH have partial evidence but are NOT flagged as gaps.
- **`evidence_type_coverage`** shows exactly which evidence types have matching artifacts and which are empty arrays. This drives the "Evidence Types Missing" column in the validation report.

---

## Step 5 — LLM Generates Narratives

**Scripts:** `orchestrate_llm.py` + F5 `SKILL.md`

This step is split into three parts:

### Step 5a — Pre-LLM (orchestrate_llm.py)

F6 formats the mapped evidence into a structured prompt for the LLM. F6 never calls the LLM directly — it produces the prompt string and returns it to F5.

### Step 5b — LLM Reasoning (F5 SKILL.md)

F5 takes the formatted prompt and sends it to the LLM. The LLM generates:

- A 3–5 sentence narrative per control with inline citations
- A confidence tier per control (HIGH / MEDIUM / LOW / GAP)
- Explicit `[GAP]` flags for controls with no evidence
- YAML front matter for the output document

### Step 5c — Post-LLM (orchestrate_llm.py)

F6 parses the raw LLM Markdown response into a structured `LLMResponse` object.

### Expected LLM Output Format

```markdown
## AC — Access Control
**Confidence**: HIGH

The repository demonstrates a structured approach to access control.
Authentication is handled through OAuth2 configuration
[auth/oauth_config.yaml — OAuth2 provider configuration] with multi-factor
authentication enforced [auth/mfa_settings.json — MFA settings].
Role-based access is defined through explicit mappings
[auth/rbac_roles.yaml — RBAC role definitions] supported by a granular
permissions matrix [auth/rbac_permissions_matrix.json — permissions per role].
Session management is configured with timeout and invalidation policies
[auth/session_config.yaml — session lifecycle configuration].

---

## DQ — Data Quality
**Confidence**: MEDIUM

[Narrative with partial evidence citations...]

> ⚠️ GAP: data_transform_test — no matching evidence found
```

### Parsing Contract

- Each control section starts with `## [CONTROL_ID] — [Control Name]`
- Confidence tier appears on the line starting with `**Confidence**:`
- Valid confidence values: `HIGH`, `MEDIUM`, `LOW`, `GAP`
- Sections are separated by `---`
- Citations follow `[file_path — description]` format

### Output — LLMResponse

```json
{
  "status": "success",
  "narratives": {
    "AC": {
      "confidence": "HIGH",
      "narrative_text": "The repository demonstrates a structured approach...",
      "artifacts_cited": ["ART-001", "ART-002", "ART-003", "ART-004", "ART-005"],
      "tests_cited": ["TST-001", "TST-002", "TST-003"],
      "evidence_types_covered": ["auth_config", "rbac_policy", "session_management"],
      "gap_flag": false
    },
    "DQ": {
      "confidence": "MEDIUM",
      "narrative_text": "Input validation is supported by...",
      "artifacts_cited": ["ART-012", "ART-013"],
      "tests_cited": ["TST-007"],
      "evidence_types_covered": ["input_validation_rule"],
      "gap_flag": false
    }
  },
  "yaml_front_matter": {
    "title": "RCSA Control Narrative Assessment",
    "generated": "2026-05-07T14:30:00Z",
    "controls_assessed": 4,
    "controls_passing": 2,
    "controls_low_confidence": 1,
    "controls_gap": 1,
    "framework": "Custom Demo Controls"
  }
}
```

### If the LLM Fails

F6 still runs. `orchestrate_llm.py` returns a degraded `LLMResponse`:

```json
{
  "status": "degraded",
  "reason": "LLM unavailable — raw evidence provided without narratives",
  "narratives": {},
  "yaml_front_matter": {
    "title": "RCSA Control Narrative Assessment — DEGRADED MODE",
    "mode": "degraded"
  },
  "raw_markdown": null,
  "mapped_evidence_passthrough": { "...": "full MappedEvidence object" }
}
```

### LLM Deviation Handling

| Issue | What F6 Does |
| ----- | ------------ |
| Missing section for a control | That control is marked as degraded, others are unaffected |
| Missing confidence line | Default to `"UNAVAILABLE"` |
| Malformed citations | Caught by `validate_citations.py` in Step 6 |
| Extra content outside control sections | Ignored |

---

## Step 6 — Validate Citations

**Script:** `validate_citations.py`

F6 takes the LLM's draft narratives and checks every inline citation against the artifact registry. This is the anti-hallucination gate.

### Citation Parsing Logic

1. Regex pattern: `\[([^\]]+?)\s—\s([^\]]+?)\]`
2. Group 1 = `file_path` (the machine-readable part)
3. Group 2 = `description` (logged but not validated for accuracy)
4. Look up `file_path` in `registry.file_path_index`
5. If found → resolved. If not found → unresolved (hallucinated).

### Output — CitationValidationResult

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

> **If unresolved citations are found:** They are flagged with an inline marker: `⚠️ [ART-999] (unresolved)`. The pipeline completes — hallucinated citations are flagged, not silently removed.

---

## Step 7 — Assemble Final Output

**Script:** `write_output.py`

F6 reads the `LLMResponse`, `CitationValidationResult`, and `MappedEvidence`, then populates F4's templates to produce the two final deliverables.

### Output 1 — `rcsa_control_narratives.md`

```markdown
---
title: "RCSA Control Narrative Assessment"
generated: "2026-05-07T14:30:00Z"
skill_version: "1.0"
total_controls: 4
overall_confidence: "MEDIUM"
---

## Summary

| Control ID | Control Name | Confidence | Artifacts | Tests | Status |
|------------|-------------|------------|-----------|-------|--------|
| AC | Access Control | HIGH | 5 | 3 | ✅ Assessed |
| CM | Change Management | HIGH | 6 | 3 | ✅ Assessed |
| DQ | Data Quality | MEDIUM | 2 | 1 | ⚠️ Partial Evidence |
| IH | Incident Handling | LOW | 2 | 1 | ⚠️ Weak Evidence |

---

## AC — Access Control

**Confidence**: HIGH
**Artifacts Cited**: 5
**Tests Cited**: 3
**Evidence Types Covered**: auth_config, rbac_policy, session_management

The repository demonstrates a structured approach to access control.
Authentication is handled through OAuth2 configuration
[auth/oauth_config.yaml — OAuth2 provider configuration] with multi-factor
authentication enforced [auth/mfa_settings.json — MFA settings].
Role-based access is defined through explicit mappings
[auth/rbac_roles.yaml — RBAC role definitions] supported by a granular
permissions matrix [auth/rbac_permissions_matrix.json — permissions per role].
Session management is configured with timeout and invalidation policies
[auth/session_config.yaml — session lifecycle configuration].

---

## DQ — Data Quality

**Confidence**: MEDIUM
**Artifacts Cited**: 2
**Tests Cited**: 1
**Evidence Types Covered**: input_validation_rule

Input validation is supported by a JSON Schema
[data/input_schema.json — required fields and constraints] and business
rule definitions [data/validation_rules.yaml — range checks and format
requirements]. However, no data transformation tests were found.

> ⚠️ GAP: data_transform_test — no matching evidence found

---

## Confidence Legend

- **HIGH** — ≥2 artifacts AND ≥1 test mapped, 0 hallucinated citations
- **MEDIUM** — ≥1 artifact OR ≥1 test mapped, 0 hallucinated citations
- **LOW** — Any evidence exists but coverage is weak or hallucinated citations detected
- **GAP** — No evidence mapped to this control
```

### Output 2 — `validation_report.md`

```markdown
---
title: "Validation Report"
generated: "2026-05-07T14:30:00Z"
total_citations: 15
valid_citations: 15
invalid_citations: 0
resolution_rate: "100%"
---

# Validation Report

## Citation Resolution

| Citation | Artifact ID | File Path | Resolved | Control |
|----------|-------------|-----------|----------|---------|
| auth/oauth_config.yaml — OAuth2 provider... | ART-001 | auth/oauth_config.yaml | ✅ Yes | AC |
| auth/mfa_settings.json — MFA settings... | ART-002 | auth/mfa_settings.json | ✅ Yes | AC |

**Total Citations**: 15 | **Resolved**: 15 | **Unresolved**: 0 | **Resolution Rate**: 100%

## Evidence Coverage

| Control ID | Artifacts | Tests | Evidence Types Covered | Evidence Types Missing |
|------------|-----------|-------|----------------------|----------------------|
| AC | 5 | 3 | auth_config, rbac_policy, session_management | (none) |
| CM | 6 | 3 | change_approval_record, deployment_config, rollback_procedure | (none) |
| DQ | 2 | 1 | input_validation_rule | data_transform_test |
| IH | 2 | 1 | monitoring_config | incident_response_plan, postmortem_record |

## Flagged for Human Review

- DQ: Missing evidence type `data_transform_test`
- IH: Missing evidence type `incident_response_plan`
- IH: Missing evidence type `postmortem_record`
```

---

# Every File in the Pipeline

## Files the User Provides

| File | What It Is | Required Keys |
| ---- | ---------- | ------------- |
| `sample_control_library.json` | 4 compliance controls, 11 evidence types | `_meta`, `controls[]`, `controls[].id`, `controls[].name`, `controls[].objective`, `controls[].evidence_types[]` |
| `sample_artifact_index.json` | Repository artifacts (10–20 for demo) | `_meta`, `artifacts[]`, `artifacts[].id`, `artifacts[].file_path`, `artifacts[].file_type`, `artifacts[].description` |
| `sample_test_catalog.json` | Test files (5–10 for demo) | `_meta`, `tests[]`, `tests[].id`, `tests[].file_path`, `tests[].file_type`, `tests[].description`, `tests[].controls_relevant[]` |

## Internal Data Structures (In-Memory at Runtime)

Unlike Skill 1, Skill 2 does NOT persist intermediate files to disk. All data structures are passed in-memory between scripts via the agent runtime.

| Data Structure | Created By | Read By | What It Contains |
| -------------- | ---------- | ------- | ---------------- |
| `ValidationResult` | `validate_input.py` | F5 (SKILL.md) | Pass/fail status per file, with specific error messages |
| `ArtifactRegistry` | `build_registry.py` | `map_evidence.py`, `validate_citations.py` | Unified lookup of all artifacts and tests, keyed by ID and by file_path |
| `MappedEvidence` | `map_evidence.py` | `orchestrate_llm.py`, `write_output.py` | Per-control evidence mapping with coverage analysis and gap flags |
| Formatted LLM Prompt | `orchestrate_llm.py` (pre-LLM) | F5 (SKILL.md) → LLM | Structured prompt string — F5 must not modify it |
| Raw LLM Response | F5 (SKILL.md) ← LLM | `orchestrate_llm.py` (post-LLM) | Markdown with control sections, confidence tiers, and citations |
| `LLMResponse` | `orchestrate_llm.py` (post-LLM) | `validate_citations.py`, `write_output.py` | Parsed narratives keyed by control ID |
| `CitationValidationResult` | `validate_citations.py` | `write_output.py` | Per-citation resolution status and counts |

## Final Deliverables (Created at Runtime by F6)

| File | Created By | What It Is |
| ---- | ---------- | ---------- |
| `output/rcsa_control_narratives.md` | `write_output.py` | Control narratives with citations, confidence tiers, and gap flags |
| `output/validation_report.md` | `write_output.py` | Citation resolution stats, evidence coverage, and flagged items |

## Static Asset Files (F4 — Checked Into Repo)

| File | What It Is | Who Uses It |
| ---- | ---------- | ----------- |
| `assets/templates/rcsa_control_narratives_template.md` | Blueprint for the control narratives output | `write_output.py` reads it at runtime |
| `assets/templates/validation_report_template.md` | Blueprint for the validation report output | `write_output.py` reads it at runtime |
| `assets/sample-data/sample_control_library.json` | 4-control, 11-evidence-type control library | Used for end-to-end testing |
| `assets/sample-data/sample_artifact_index.json` | 15-artifact demo-scale artifact index | Used for end-to-end testing |
| `assets/sample-data/sample_test_catalog.json` | 8-test demo-scale test catalog | Used for end-to-end testing |
| `assets/citation_format.md` | Citation format specification and resolution rules | Referenced by F5 and F6 |

---

# Key Rules That Govern the Whole Skill

## Confidence Tiers

Confidence is always one of exactly four values (plus one fallback). These are the only allowed strings.

| Value | Meaning | When It's Assigned |
| ----- | ------- | ------------------ |
| `HIGH` | Strong evidence across all evidence types | ≥2 artifacts AND ≥1 test mapped, 0 hallucinated citations |
| `MEDIUM` | Partial evidence — some types covered | ≥1 artifact OR ≥1 test mapped, 0 hallucinated citations |
| `LOW` | Weak evidence — minimal coverage | Any evidence exists but hallucinated citations detected |
| `GAP` | No evidence found | Zero artifacts AND zero tests mapped to this control |
| `UNAVAILABLE` | LLM did not assign a tier | F6 default when tier is missing or LLM is unavailable |

> **Rule:** A GAP flag means zero evidence. It is NOT the same as LOW confidence. LOW means some evidence exists but it's weak. GAP means nothing was found.

## Citation Format

All citations in the narratives follow this exact format:

- **Syntax:** `[file_path — description]`
- **Separator:** ` — ` (space, em dash, space)
- **Regex:** `\[([^\]]+?)\s—\s([^\]]+?)\]`
- **Multi-file evidence:** Use separate citations per file — no sub-index syntax
- **Example:** `[auth/oauth_config.yaml — OAuth2 provider configuration]`

> **If F4 changes the citation format:** F5's SKILL.md must be updated, F6's `validate_citations.py` regex must be updated, and all three features must be re-aligned.

## Gap Handling

The system MUST never imply compliance without proof (Constitution Principle 1).

- If `mapped_artifacts` is empty AND `mapped_tests` is empty → GAP flag
- Gap flags must have **zero false negatives**
- Gap callout format: `> ⚠️ GAP: {evidence_type} — no matching evidence found`
- The LLM flags gaps only — it does NOT suggest remediation steps

## Placeholder Values — Degraded Mode

When the LLM fails, F6 fills these values and the pipeline continues:

| Field | Degraded Value |
| ----- | -------------- |
| `status` | `"degraded"` |
| `narratives` | `{}` (empty object) |
| `yaml_front_matter.mode` | `"degraded"` |
| `yaml_front_matter.title` | `"RCSA Control Narrative Assessment — DEGRADED MODE"` |
| `raw_markdown` | `null` |
| `mapped_evidence_passthrough` | Full MappedEvidence object for manual review |

In degraded mode, `write_output.py` produces narratives populated with raw mapped evidence only and validation report noting "LLM unavailable — raw evidence mode."

## Template Authority

F4's templates are the single source of truth for output format. F6 scripts read the templates and follow their structure — they never hardcode section names or column order. All placeholders use `{{field_name}}` double curly brace syntax.

> **A missing template is not graceful degradation — it's a setup error.** If a template file is missing, the pipeline stops with a clear error.

## Graceful Degradation

If the LLM goes offline, F6 still runs. Pre-LLM scripts function independently of LLM availability. The pipeline ALWAYS produces output files. The only exception is input validation failure.

| Scenario | Who Detects | What Happens | Continues? |
| -------- | ----------- | ------------ | ---------- |
| Input validation fails | F6 (`validate_input.py`) | Returns `status: "fail"` with errors | ❌ No |
| LLM times out | F5 | F5 passes empty/error to `orchestrate_llm.py` | ✅ Degraded |
| LLM returns empty | F5 | F5 passes empty string to `orchestrate_llm.py` | ✅ Degraded |
| LLM returns malformed output | F6 (`orchestrate_llm.py`) | Parses what it can, degrades the rest | ✅ Partial |
| LLM hallucinates citations | F6 (`validate_citations.py`) | Flags unresolved citations in report | ✅ Reported |
| One control section missing | F6 (`orchestrate_llm.py`) | That control marked degraded, others normal | ✅ Per-control |

## Error Signaling Between F5 and F6

### F6 → F5 (Script errors)

| Signal | Meaning | F5 Action |
| ------ | ------- | --------- |
| Exit code 0 + `status: "pass"` or `"success"` | Script succeeded | Proceed to next step |
| Exit code 0 + `status: "fail"` | Validation failed (Step 2 only) | Stop pipeline, report errors |
| Exit code 0 + `status: "degraded"` | LLM failed but pipeline can continue | Proceed in degraded mode |
| Exit code 1 | Unexpected script error (bug) | Stop pipeline, report error |

### F5 → F6 (LLM errors)

| Signal | Meaning | F6 Action |
| ------ | ------- | --------- |
| Valid Markdown string | LLM succeeded | Parse normally |
| Empty string `""` | LLM returned nothing | Return degraded `LLMResponse` |
| Error flag / null | LLM timed out or errored | Return degraded `LLMResponse` |

## Performance Budget

| Owner | Component | Budget |
| ----- | --------- | ------ |
| F6 | All deterministic scripts | < 30 seconds total |
| F5 | LLM invocation (per control) | ≤ 30 seconds |
| F5 | LLM invocation (all controls) | ≤ 120 seconds |
| F6 | `orchestrate_llm.py` formatting + parsing | < 5 seconds |
| **Total** | **Full pipeline** | **< 155 seconds worst case** |

## Sample Data Coverage Design

F4 sample data is intentionally designed with specific coverage patterns to test gap detection:

| Control | Evidence Provided | Intentional Gaps |
| ------- | ----------------- | ---------------- |
| **AC** | All 3 types (5 artifacts, 3 tests) | None — full coverage |
| **CM** | All 3 types (6 artifacts, 3 tests) | None — full coverage |
| **DQ** | `input_validation_rule` (2 artifacts, 1 test) | `data_transform_test` missing |
| **IH** | `monitoring_config` (2 artifacts, 1 test) | `incident_response_plan` and `postmortem_record` missing |

## Demo-Scale Constraints

- 15 artifacts in sample artifact index
- 8 tests in sample test catalog
- 4 controls in control library (AC, CM, DQ, IH)
- 11 evidence types total
- < 60 seconds for deterministic pipeline portions
- Python 3.11 standard library only — zero third-party dependencies
- All files UTF-8 encoded
- No PII, secrets, or real Synchrony identifiers in sample data

---

# Cross-Feature Dependencies

| Upstream | Downstream | What's Shared | Contract |
| -------- | ---------- | ------------- | -------- |
| F4 (Assets) | F5 (SKILL.md) | Asset file paths, YAML front matter fields, template structure, citation format syntax | Directory structure and templates locked in F4 plan |
| F4 (Assets) | F6 (Scripts) | Control library JSON schema, artifact index + test catalog JSON schemas, template placeholder syntax, sample data | JSON schemas and placeholder format locked in F4 plan |
| F5 (SKILL.md) | F6 (Scripts) | LLM response structure, script invocation sequence, confidence tier values | Cross-feature contract defined in F5 and F6 plans |

## Cross-Feature Re-validation Gates

1. **After F5 build** → Verify F5's LLM output instructions match F4's template fields and citation format. Reconcile mismatches.
2. **After F6 plan approval** → Verify F6's input expectations match F4's sample data and JSON schemas. **Blocking gate** — F6 implementation cannot start until this passes.

---

# Feature Ownership Summary

| What | Who Owns It | Where to Find More |
| ---- | ----------- | ------------------ |
| Output templates (narratives + validation report) | F4 | `specs/f4-rcsa-assets/plan.md` |
| Sample input data (3 JSON files) | F4 | `specs/f4-rcsa-assets/plan.md` |
| Citation format specification | F4 | `specs/f4-rcsa-assets/plan.md` |
| Control library (4 controls, 11 evidence types) | F4 | `specs/f4-rcsa-assets/plan.md` |
| LLM orchestration instructions | F5 | `specs/005-skill-rcsa/plan.md` |
| Confidence tier rubric | F5 | `specs/005-skill-rcsa/plan.md` |
| Gap detection logic (LLM reasoning) | F5 | `specs/005-skill-rcsa/plan.md` |
| Narrative generation instructions | F5 | `specs/005-skill-rcsa/plan.md` |
| Input validation (3 JSON files) | F6 | `specs/f6-rcsa-scripts/plan.md` |
| Artifact registry construction | F6 | `specs/f6-rcsa-scripts/plan.md` |
| Evidence-to-control mapping | F6 | `specs/f6-rcsa-scripts/plan.md` |
| LLM prompt formatting + response parsing | F6 | `specs/f6-rcsa-scripts/plan.md` |
| Citation validation (anti-hallucination) | F6 | `specs/f6-rcsa-scripts/plan.md` |
| Template population + file writing | F6 | `specs/f6-rcsa-scripts/plan.md` |
| Graceful degradation logic | F6 | `specs/f6-rcsa-scripts/plan.md` |
| All data model schemas | F6 | `specs/f6-rcsa-scripts/data-model.md` |
| F5↔F6 interface contract | F5 + F6 | `specs/005-skill-rcsa/contracts.md` |
| F6↔F4 interface contract | F4 + F6 | `specs/f6-rcsa-scripts/contracts/` |
| Full repository layout | Project spec | `project-spec.md` |

---

*This document is the starting point for understanding the RCSA Control Narrative skill. It does not replace the feature-level plans — it points you to them. If something here conflicts with a feature plan, the feature plan wins.*
