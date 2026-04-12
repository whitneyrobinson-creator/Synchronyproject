# F5 — SKILL.md: RCSA Control Narrative Generation — Quickstart

**Feature Branch**: `f5-rcsa-skill`
**Created**: 2026-04-10
**Updated**: 2026-04-12
**Status**: Draft
**Phase**: 1

---

## Overview

This guide explains how to run and test the F5 SKILL.md in Claude. By the end, you'll have generated two output documents from sample data and verified they meet the spec.

**Control ID format:** Control IDs are uppercase 2–4 character strings (e.g., `AC`, `CM`, `DQ`, `IH`), as defined by F4's `control_library.schema.json`.

---

## Prerequisites

1. **Claude access** — claude.ai or Claude Desktop
2. **F6 scripts available** — the SKILL.md calls these as tools during orchestration
3. **Sample data** — located in `.cursor/skills/rcsa/assets/sample-data/`

---

## Step 1: Prepare Sample Inputs

You need three input files. For the demo, use the sample data provided by F4/F6:

### control_library.yaml

```yaml
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

### artifact_registry.yaml

Built by F6's `build_registry.py`. For testing, use the sample registry that includes artifacts for AC and CM but none for DQ or IH (to test GAP detection).

### mapped_evidence.yaml

Built by F6's `map_evidence.py`. For testing, use the sample mapping that:
- Maps 3 artifacts to AC (direct relevance)
- Maps 1 artifact to CM (direct relevance)
- Maps 0 artifacts to DQ (empty — triggers GAP)
- Maps 0 artifacts to IH (empty — triggers GAP)

---

## Step 2: Run the SKILL.md

1. Open Claude
2. Load the SKILL.md file (`.cursor/skills/rcsa/SKILL.md`)
3. Provide the three input files
4. The SKILL.md will orchestrate the 6-step workflow:

| Step | Action | Type |
|------|--------|------|
| 1 | Validate inputs | F6 script |
| 2 | Build artifact registry | F6 script |
| 3 | Map evidence to controls | F6 script |
| 4 | Generate narratives | LLM reasoning |
| 5 | Validate citations | F6 script |
| 6 | Assemble final output | LLM reasoning |

---

## Step 3: Verify Output

The SKILL.md should produce two files:

### Output 1: `rcsa_control_narratives.md`

**Check the YAML front matter:**

```yaml
---
title: "RCSA Control Narrative Assessment"
generated: "<timestamp>"
controls_assessed: 4
controls_passing: 1
controls_low_confidence: 1
controls_gap: 2
framework: "Custom Demo Controls"
---
```

**Check the summary table:**

| Control ID | Control Name | Confidence | Artifacts | Status |
|------------|-------------|------------|-----------|--------|
| AC | Access Control | HIGH | 3 | ✅ Assessed |
| CM | Change Management | LOW | 1 | ⚠️ Weak Evidence |
| DQ | Data Quality | GAP | 0 | 🔴 No Evidence |
| IH | Incident Handling | GAP | 0 | 🔴 No Evidence |

**Check per-control sections:**

- [ ] **AC section**: 3–5 sentences, 3 inline citations, confidence = HIGH
- [ ] **CM section**: 3–5 sentences, 1 inline citation, confidence = LOW
- [ ] **DQ section**: `[GAP]` flag, no citations, confidence = GAP
- [ ] **IH section**: `[GAP]` flag, no citations, confidence = GAP

**Check citation format:**
- Every citation follows `[file_path — description]`
- Example: `[src/auth/login.py — credential validation and session token generation]`

### Output 2: `validation_report.md`

**Check citation resolution:**

- [ ] All citations resolve to real artifacts in the registry
- [ ] Resolution rate = 100% (no hallucinated citations)
- [ ] Each citation shows artifact ID, file path, and match type

**Check confidence scoring audit:**

| Control ID | Confidence | Direct | Indirect | Rubric Match |
|------------|------------|--------|----------|--------------|
| AC | HIGH | 3 | 0 | ✅ ≥2 direct → HIGH |
| CM | LOW | 1 | 0 | ✅ 1 direct, 0 indirect → LOW |
| DQ | GAP | 0 | 0 | ✅ 0 artifacts → GAP |
| IH | GAP | 0 | 0 | ✅ 0 artifacts → GAP |

---

## Step 4: Verification Checklist

Run through this checklist after each test:

### Accuracy
- [ ] Zero false negatives on GAP detection (DQ and IH must be flagged)
- [ ] No hallucinated citations (every cited file exists in the registry)
- [ ] Confidence tiers match the rubric

### Structure
- [ ] YAML front matter is valid and parseable
- [ ] Summary table has all 4 controls
- [ ] Per-control sections follow the defined layout
- [ ] Validation report has all required sections

### Edge Cases
- [ ] Controls with zero evidence get `[GAP]` flag (not a weak narrative)
- [ ] Controls with 1 artifact get LOW confidence (not MEDIUM or HIGH)
- [ ] Citations reference exact file paths from the registry (no modifications)

---

## Troubleshooting

| Problem | Likely Cause | Fix |
|---------|-------------|-----|
| SKILL.md stops at Step 1 | Input validation failed | Check F6's error output — likely a malformed control library |
| Missing control in output | Control not in control_library.yaml | Verify all 4 controls are in the input |
| Wrong confidence tier | Rubric mismatch | Check artifact count and relevance against the confidence rubric |
| Hallucinated citation | LLM cited a file not in the registry | Re-run — if persistent, tighten the SKILL.md citation instructions |
| No GAP flag on empty control | SKILL.md reasoning error | Verify mapped_evidence shows empty `mapped_artifacts` for that control |
| Control ID format wrong | Old `AC-001` style instead of `AC` | Ensure control_library.yaml uses short-form IDs per F4 schema |

---

## Running Multiple Times

To test structural consistency (SC-004 target: ≥90%):

1. Run the same inputs 5 times
2. Compare output structure across runs (YAML fields, table columns, section headings)
3. Narrative wording will vary — that's expected
4. Structure should be identical or near-identical

---

*This quickstart uses the demo control set (AC, CM, DQ, IH). For production use, replace the control library with your target framework's controls.*
