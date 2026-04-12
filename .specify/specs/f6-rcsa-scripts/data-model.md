# Data Model: F6 — RCSA Scripts (Deterministic Python Pipeline)

**Feature**: F6 — RCSA Scripts | **Date**: 2026-04-12 | **Status**: Draft

---

## Overview

This document defines the exact data structures that flow into, through, and out of each F6 script. Every schema is derived from F4's locked sample data files and F5's SKILL.md contracts.

**Data flow summary:**

```
User provides 3 JSON files
        │
        ▼
validate_input.py ──→ ValidationResult
        │
        ▼
build_registry.py ──→ ArtifactRegistry
        │
        ▼
map_evidence.py ──→ MappedEvidence
        │
        ▼
orchestrate_llm.py ──→ LLMResponse (or DegradedResponse)
        │
        ▼
validate_citations.py ──→ CitationValidationResult
        │
        ▼
write_output.py ──→ rcsa_control_narratives.md + validation_report.md
```

---

## 1. External Input Schemas (F4 — Locked)

These are the 3 JSON files the user provides. F6 reads them; F6 does not define them.

### 1.1 Control Library (`sample_control_library.json`)

```json
{
  "_meta": {
    "scale": "demo",
    "note": "Replace with production data post-handoff"
  },
  "controls": [
    {
      "id": "AC",
      "name": "Access Control",
      "objective": "Ensure system access is restricted to authorized users through authentication, authorization, and session management",
      "evidence_types": [
        {
          "id": "auth_config",
          "label": "Authentication Configuration",
          "description": "Configuration files defining authentication mechanisms such as OAuth, SSO, or MFA settings"
        },
        {
          "id": "rbac_policy",
          "label": "RBAC Policy Definition",
          "description": "Role-based access control definitions specifying roles, permissions, and access levels"
        },
        {
          "id": "session_management",
          "label": "Session Management Configuration",
          "description": "Configuration or code governing session lifecycle including timeout, renewal, and invalidation"
        }
      ]
    }
  ]
}
```

**Field definitions:**

| Field | Type | Required | Description |
|---|---|---|---|
| `_meta` | object | No | Metadata about the file. Ignored by scripts. |
| `controls` | array[object] | Yes | List of compliance controls to assess |
| `controls[].id` | string | Yes | Unique control ID. Pattern: `^[A-Z]{2,4}$`. Values: `AC`, `CM`, `DQ`, `IH` |
| `controls[].name` | string | Yes | Human-readable control name |
| `controls[].objective` | string | Yes | What the control is supposed to achieve |
| `controls[].evidence_types` | array[object] | Yes | Types of artifacts that constitute evidence for this control |
| `controls[].evidence_types[].id` | string | Yes | Unique evidence type ID (e.g., `auth_config`, `rbac_policy`) |
| `controls[].evidence_types[].label` | string | Yes | Human-readable label |
| `controls[].evidence_types[].description` | string | Yes | What this evidence type looks like |

**Demo data**: 4 controls (AC, CM, DQ, IH) with 2–3 evidence types each (11 total evidence types).

---

### 1.2 Artifact Index (`sample_artifact_index.json`)

```json
{
  "_meta": {
    "scale": "demo",
    "note": "Replace with production data post-handoff"
  },
  "artifacts": [
    {
      "id": "ART-001",
      "file_path": "auth/oauth_config.yaml",
      "file_type": "yaml",
      "description": "OAuth2 provider configuration defining authentication flows, token lifetimes, and allowed redirect URIs"
    }
  ]
}
```

**Field definitions:**

| Field | Type | Required | Description |
|---|---|---|---|
| `_meta` | object | No | Metadata. Ignored by scripts. |
| `artifacts` | array[object] | Yes | List of repository artifacts |
| `artifacts[].id` | string | Yes | Unique artifact ID. Format: `ART-NNN` |
| `artifacts[].file_path` | string | Yes | Relative path from repository root |
| `artifacts[].file_type` | string | Yes | File format (e.g., `yaml`, `json`, `python`, `markdown`, `terraform`, `shell`) |
| `artifacts[].description` | string | Yes | What this artifact contains/does |

**Demo data**: 15 artifacts across 4 directories (`auth/`, `deploy/`, `data/`, `monitoring/`).

---

### 1.3 Test Catalog (`sample_test_catalog.json`)

```json
{
  "_meta": {
    "scale": "demo",
    "note": "Replace with production data post-handoff"
  },
  "tests": [
    {
      "id": "TST-001",
      "file_path": "tests/auth/test_rbac_permissions.py",
      "file_type": "python",
      "description": "Verifies that RBAC role assignments enforce correct permission boundaries across application modules",
      "controls_relevant": ["AC"]
    }
  ]
}
```

**Field definitions:**

| Field | Type | Required | Description |
|---|---|---|---|
| `_meta` | object | No | Metadata. Ignored by scripts. |
| `tests` | array[object] | Yes | List of test files |
| `tests[].id` | string | Yes | Unique test ID. Format: `TST-NNN` |
| `tests[].file_path` | string | Yes | Relative path from repository root |
| `tests[].file_type` | string | Yes | File format (e.g., `python`, `shell`) |
| `tests[].description` | string | Yes | What this test verifies |
| `tests[].controls_relevant` | array[string] | Yes | Control IDs this test provides evidence for. Values must match `controls[].id` from the control library. |

**Demo data**: 8 tests. Distribution: AC (3 tests), CM (3 tests), DQ (1 test), IH (1 test).

---

## 2. Internal Data Schemas (F6 — Defined Here)

These are the data structures F6 scripts produce and pass between each other via the agent runtime.

### 2.1 ValidationResult — Output of `validate_input.py`

```json
{
  "status": "pass",
  "files_validated": [
    {
      "file": "control_library",
      "status": "pass",
      "errors": []
    },
    {
      "file": "artifact_index",
      "status": "pass",
      "errors": []
    },
    {
      "file": "test_catalog",
      "status": "pass",
      "errors": []
    }
  ]
}
```

**Failure example:**

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
    {
      "file": "artifact_index",
      "status": "pass",
      "errors": []
    },
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

**Field definitions:**

| Field | Type | Required | Description |
|---|---|---|---|
| `status` | string | Yes | `"pass"` or `"fail"`. `"fail"` if ANY file fails. |
| `files_validated` | array[object] | Yes | Per-file validation results |
| `files_validated[].file` | string | Yes | Which input file: `"control_library"`, `"artifact_index"`, or `"test_catalog"` |
| `files_validated[].status` | string | Yes | `"pass"` or `"fail"` for this specific file |
| `files_validated[].errors` | array[string] | Yes | Human-readable error messages. Empty if pass. |

**Validation rules applied:**

| Check | What It Validates |
|---|---|
| File existence | All 3 JSON files exist at expected paths |
| JSON parsing | All 3 files are valid JSON |
| Required fields | `controls`, `artifacts`, `tests` arrays exist and are non-empty |
| Control ID format | Every `controls[].id` matches `^[A-Z]{2,4}$` |
| No duplicate IDs | No duplicate `controls[].id`, `artifacts[].id`, or `tests[].id` |
| Evidence types non-empty | Every control has at least one evidence type |
| Test control references | Every value in `tests[].controls_relevant` matches a `controls[].id` |
| Artifact ID format | Every `artifacts[].id` matches `ART-NNN` pattern |
| Test ID format | Every `tests[].id` matches `TST-NNN` pattern |

---

### 2.2 ArtifactRegistry — Output of `build_registry.py`

```json
{
  "registry": {
    "artifacts": {
      "ART-001": {
        "id": "ART-001",
        "file_path": "auth/oauth_config.yaml",
        "file_type": "yaml",
        "description": "OAuth2 provider configuration defining authentication flows, token lifetimes, and allowed redirect URIs",
        "source": "artifact_index"
      },
      "ART-002": {
        "id": "ART-002",
        "file_path": "auth/mfa_settings.json",
        "file_type": "json",
        "description": "Multi-factor authentication settings including enforced methods and bypass rules",
        "source": "artifact_index"
      }
    },
    "tests": {
      "TST-001": {
        "id": "TST-001",
        "file_path": "tests/auth/test_rbac_permissions.py",
        "file_type": "python",
        "description": "Verifies that RBAC role assignments enforce correct permission boundaries across application modules",
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

**Field definitions:**

| Field | Type | Required | Description |
|---|---|---|---|
| `registry.artifacts` | object | Yes | Artifact entries keyed by ID for O(1) lookup |
| `registry.tests` | object | Yes | Test entries keyed by ID for O(1) lookup |
| `registry.file_path_index` | object | Yes | Reverse lookup: file_path → ID. Used by `validate_citations.py` to resolve citations. |
| `registry.stats` | object | Yes | Summary counts for logging and reporting |

**Why this structure:**
- `artifacts` and `tests` are keyed by ID (not arrays) for fast lookup during evidence mapping
- `file_path_index` is the critical structure for citation validation — when the LLM cites `[auth/oauth_config.yaml — ...]`, F6 looks up the file path here to confirm it exists
- `source` field tracks whether an entry came from the artifact index or test catalog

---

### 2.3 MappedEvidence — Output of `map_evidence.py`

```json
{
  "mappings": [
    {
      "control_id": "AC",
      "control_name": "Access Control",
      "control_objective": "Ensure system access is restricted to authorized users through authentication, authorization, and session management",
      "mapped_artifacts": [
        {
          "id": "ART-001",
          "file_path": "auth/oauth_config.yaml",
          "file_type": "yaml",
          "description": "OAuth2 provider configuration defining authentication flows, token lifetimes, and allowed redirect URIs",
          "evidence_type_matched": "auth_config",
          "source": "artifact_index"
        },
        {
          "id": "ART-002",
          "file_path": "auth/mfa_settings.json",
          "file_type": "json",
          "description": "Multi-factor authentication settings including enforced methods and bypass rules",
          "evidence_type_matched": "auth_config",
          "source": "artifact_index"
        },
        {
          "id": "ART-003",
          "file_path": "auth/rbac_roles.yaml",
          "file_type": "yaml",
          "description": "Role-based access control definitions mapping roles to permissions across application modules",
          "evidence_type_matched": "rbac_policy",
          "source": "artifact_index"
        },
        {
          "id": "ART-004",
          "file_path": "auth/rbac_permissions_matrix.json",
          "file_type": "json",
          "description": "Permissions matrix specifying granular access levels per role and resource type",
          "evidence_type_matched": "rbac_policy",
          "source": "artifact_index"
        },
        {
          "id": "ART-005",
          "file_path": "auth/session_config.yaml",
          "file_type": "yaml",
          "description": "Session management configuration including timeout duration, renewal policy, and invalidation triggers",
          "evidence_type_matched": "session_management",
          "source": "artifact_index"
        }
      ],
      "mapped_tests": [
        {
          "id": "TST-001",
          "file_path": "tests/auth/test_rbac_permissions.py",
          "description": "Verifies that RBAC role assignments enforce correct permission boundaries across application modules",
          "source": "test_catalog"
        },
        {
          "id": "TST-002",
          "file_path": "tests/auth/test_session_timeout.py",
          "description": "Validates that user sessions expire after the configured timeout period and cannot be reused after invalidation",
          "source": "test_catalog"
        },
        {
          "id": "TST-003",
          "file_path": "tests/auth/test_oauth_flow.py",
          "description": "Tests the OAuth2 authentication flow including token issuance, refresh, and revocation",
          "source": "test_catalog"
        }
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
      "control_objective": "Ensure all changes to production systems follow approved processes with rollback capability",
      "mapped_artifacts": [
        {
          "id": "ART-006",
          "file_path": "deploy/change_approval_workflow.yaml",
          "file_type": "yaml",
          "description": "CI/CD pipeline stage requiring manual approval before production deployment",
          "evidence_type_matched": "change_approval_record",
          "source": "artifact_index"
        },
        {
          "id": "ART-007",
          "file_path": "deploy/pr_review_policy.md",
          "file_type": "markdown",
          "description": "Pull request review policy requiring minimum reviewers and passing checks before merge",
          "evidence_type_matched": "change_approval_record",
          "source": "artifact_index"
        },
        {
          "id": "ART-008",
          "file_path": "deploy/ci_pipeline.yaml",
          "file_type": "yaml",
          "description": "CI/CD pipeline definition including build, test, staging, and production deployment stages",
          "evidence_type_matched": "deployment_config",
          "source": "artifact_index"
        },
        {
          "id": "ART-009",
          "file_path": "deploy/terraform_main.tf",
          "file_type": "terraform",
          "description": "Infrastructure-as-code defining production environment resources and deployment configuration",
          "evidence_type_matched": "deployment_config",
          "source": "artifact_index"
        },
        {
          "id": "ART-010",
          "file_path": "deploy/rollback_runbook.md",
          "file_type": "markdown",
          "description": "Step-by-step rollback procedure for reverting failed production deployments",
          "evidence_type_matched": "rollback_procedure",
          "source": "artifact_index"
        },
        {
          "id": "ART-011",
          "file_path": "deploy/rollback_script.sh",
          "file_type": "shell",
          "description": "Automated rollback script that reverts the last production deployment to the previous known-good state",
          "evidence_type_matched": "rollback_procedure",
          "source": "artifact_index"
        }
      ],
      "mapped_tests": [
        {
          "id": "TST-004",
          "file_path": "tests/deploy/test_approval_gate.py",
          "description": "Verifies that the CI/CD pipeline blocks production deployment without required approvals",
          "source": "test_catalog"
        },
        {
          "id": "TST-005",
          "file_path": "tests/deploy/test_rollback.sh",
          "description": "End-to-end test that triggers a rollback and verifies the system returns to the previous known-good state",
          "source": "test_catalog"
        },
        {
          "id": "TST-006",
          "file_path": "tests/deploy/test_deployment_pipeline.py",
          "description": "Validates that the deployment pipeline executes all required stages in order including build, test, and staging",
          "source": "test_catalog"
        }
      ],
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
      "control_objective": "Ensure data inputs are validated and transformations preserve integrity",
      "mapped_artifacts": [
        {
          "id": "ART-012",
          "file_path": "data/input_schema.json",
          "file_type": "json",
          "description": "JSON Schema defining required fields, types, and constraints for incoming data records",
          "evidence_type_matched": "input_validation_rule",
          "source": "artifact_index"
        },
        {
          "id": "ART-013",
          "file_path": "data/validation_rules.yaml",
          "file_type": "yaml",
          "description": "Business rule definitions for input data validation including range checks and format requirements",
          "evidence_type_matched": "input_validation_rule",
          "source": "artifact_index"
        }
      ],
      "mapped_tests": [
        {
          "id": "TST-007",
          "file_path": "tests/data/test_input_validation.py",
          "description": "Tests that invalid input records are rejected with appropriate error messages based on schema constraints",
          "source": "test_catalog"
        }
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
      "control_objective": "Ensure security incidents are detected, responded to, and reviewed",
      "mapped_artifacts": [
        {
          "id": "ART-014",
          "file_path": "monitoring/alerting_rules.yaml",
          "file_type": "yaml",
          "description": "Alert rule definitions for detecting anomalies including error rate thresholds and latency spikes",
          "evidence_type_matched": "monitoring_config",
          "source": "artifact_index"
        },
        {
          "id": "ART-015",
          "file_path": "monitoring/dashboard_config.json",
          "file_type": "json",
          "description": "Monitoring dashboard configuration displaying key system health metrics and incident indicators",
          "evidence_type_matched": "monitoring_config",
          "source": "artifact_index"
        }
      ],
      "mapped_tests": [
        {
          "id": "TST-008",
          "file_path": "tests/monitoring/test_alert_triggers.py",
          "description": "Verifies that alerting rules fire correctly when error rate and latency thresholds are exceeded",
          "source": "test_catalog"
        }
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

**Field definitions:**

| Field | Type | Required | Description |
|---|---|---|---|
| `mappings` | array[object] | Yes | One entry per control from the control library |
| `mappings[].control_id` | string | Yes | References `controls[].id` |
| `mappings[].control_name` | string | Yes | Human-readable name |
| `mappings[].control_objective` | string | Yes | Passed through for LLM context |
| `mappings[].mapped_artifacts` | array[object] | Yes | Artifacts mapped to this control. Empty = gap candidate. |
| `mappings[].mapped_artifacts[].evidence_type_matched` | string | Yes | Which evidence type this artifact satisfies |
| `mappings[].mapped_tests` | array[object] | Yes | Tests mapped via `controls_relevant`. Empty = no test evidence. |
| `mappings[].evidence_type_coverage` | object | Yes | Which evidence types have artifacts and which are empty |
| `mappings[].gap_flag` | boolean | Yes | `true` if BOTH `mapped_artifacts` AND `mapped_tests` are empty |
| `mappings[].artifact_count` | integer | Yes | Count of mapped artifacts |
| `mappings[].test_count` | integer | Yes | Count of mapped tests |
| `summary` | object | Yes | Aggregate stats for logging |
| `summary.unmapped_artifacts` | array[string] | Yes | Artifact IDs that didn't match any control's evidence types |
| `summary.unmapped_tests` | array[string] | Yes | Test IDs with `controls_relevant` referencing unknown control IDs |

**Mapping logic:**

The mapping between artifacts and controls is the core deterministic logic in F6. Here is how it works:

1. For each control, collect its `evidence_types[].id` values (e.g., AC has `auth_config`, `rbac_policy`, `session_management`)
2. For each artifact, determine which evidence type it matches based on its `description` and `file_path` context
3. For each test, read `controls_relevant[]` to directly assign it to controls
4. An artifact can map to multiple controls if it matches evidence types from more than one

**Note on artifact-to-evidence-type matching:** The artifact index does NOT contain an `evidence_type` field. The mapping must be inferred from the artifact's `description` and `file_path` against the control library's `evidence_types[].description`. This is the one step where the LLM's reasoning may be needed to assist — or F6 can implement keyword/path-based heuristics for the demo. This is a design decision to be resolved during implementation.

---

### 2.4 LLMResponse — Output of `orchestrate_llm.py`

This is the structured representation of what the LLM returns after generating narratives.

**Success case:**

```json
{
  "status": "success",
  "narratives": {
    "AC": {
      "confidence": "HIGH",
      "narrative_text": "The repository demonstrates a structured approach to access control. Authentication is handled through OAuth2 configuration defining authentication flows and token lifetimes [auth/oauth_config.yaml — OAuth2 provider configuration] with multi-factor authentication enforced as an additional layer [auth/mfa_settings.json — MFA settings including enforced methods]. Role-based access is defined through explicit role-to-permission mappings [auth/rbac_roles.yaml — RBAC role definitions] supported by a granular permissions matrix [auth/rbac_permissions_matrix.json — permissions per role and resource type]. Session management is configured with timeout and invalidation policies [auth/session_config.yaml — session lifecycle configuration].",
      "artifacts_cited": ["ART-001", "ART-002", "ART-003", "ART-004", "ART-005"],
      "tests_cited": ["TST-001", "TST-002", "TST-003"],
      "evidence_types_covered": ["auth_config", "rbac_policy", "session_management"],
      "gap_flag": false
    },
    "DQ": {
      "confidence": "GAP",
      "narrative_text": "[GAP] No artifacts were mapped to the Data Quality control...",
      "artifacts_cited": [],
      "tests_cited": [],
      "evidence_types_covered": [],
      "gap_flag": true
    }
  },
  "yaml_front_matter": {
    "title": "RCSA Control Narrative Assessment",
    "generated": "2026-04-12T14:30:00Z",
    "controls_assessed": 4,
    "controls_passing": 2,
    "controls_low_confidence": 1,
    "controls_gap": 1,
    "framework": "Custom Demo Controls"
  },
  "raw_markdown": "---\ntitle: RCSA Control Narrative Assessment\n..."
}
```

**Degraded case (LLM unavailable):**

```json
{
  "status": "degraded",
  "reason": "LLM unavailable — raw evidence provided without narratives",
  "narratives": {},
  "yaml_front_matter": {
    "title": "RCSA Control Narrative Assessment — DEGRADED MODE",
    "generated": "2026-04-12T14:30:00Z",
    "controls_assessed": 4,
    "controls_passing": 0,
    "controls_low_confidence": 0,
    "controls_gap": 0,
    "framework": "Custom Demo Controls",
    "mode": "degraded"
  },
  "raw_markdown": null,
  "mapped_evidence_passthrough": { "...": "full MappedEvidence object passed through for manual review" }
}
```

**Field definitions:**

| Field | Type | Required | Description |
|---|---|---|---|
| `status` | string | Yes | `"success"` or `"degraded"` |
| `reason` | string | If degraded | Why the LLM failed |
| `narratives` | object | Yes | Keyed by control ID. Empty object if degraded. |
| `narratives[control_id].confidence` | string | Yes | `"HIGH"`, `"MEDIUM"`, `"LOW"`, or `"GAP"`. Assigned by LLM. |
| `narratives[control_id].narrative_text` | string | Yes | The 3–5 sentence narrative with inline citations |
| `narratives[control_id].artifacts_cited` | array[string] | Yes | Artifact IDs referenced in the narrative |
| `narratives[control_id].tests_cited` | array[string] | Yes | Test IDs referenced in the narrative |
| `narratives[control_id].gap_flag` | boolean | Yes | `true` if this control has no evidence |
| `yaml_front_matter` | object | Yes | Metadata for the output document header |
| `raw_markdown` | string | If success | The complete Markdown output from the LLM |
| `mapped_evidence_passthrough` | object | If degraded | The full MappedEvidence for manual review |

---

### 2.5 CitationValidationResult — Output of `validate_citations.py`

```json
{
  "total_citations": 8,
  "resolved": 8,
  "unresolved": 0,
  "resolution_rate": 1.0,
  "citations": [
    {
      "citation_text": "auth/oauth_config.yaml — OAuth2 provider configuration",
      "file_path_extracted": "auth/oauth_config.yaml",
      "resolved_to": "ART-001",
      "resolved": true,
      "control_id": "AC"
    },
    {
      "citation_text": "nonexistent/file.py — some description",
      "file_path_extracted": "nonexistent/file.py",
      "resolved_to": null,
      "resolved": false,
      "control_id": "CM"
    }
  ],
  "unresolved_list": [
    {
      "citation_text": "nonexistent/file.py — some description",
      "file_path_extracted": "nonexistent/file.py",
      "control_id": "CM"
    }
  ]
}
```

**Field definitions:**

| Field | Type | Required | Description |
|---|---|---|---|
| `total_citations` | integer | Yes | Total citations found in the LLM output |
| `resolved` | integer | Yes | Citations that match a file path in the registry |
| `unresolved` | integer | Yes | Citations that do NOT match any file path |
| `resolution_rate` | float | Yes | `resolved / total_citations`. 0.0 if no citations. |
| `citations` | array[object] | Yes | Per-citation validation detail |
| `citations[].citation_text` | string | Yes | The full citation as it appears in the narrative |
| `citations[].file_path_extracted` | string | Yes | The file path portion extracted from the citation |
| `citations[].resolved_to` | string or null | Yes | The artifact/test ID it resolved to, or null |
| `citations[].resolved` | boolean | Yes | Whether the citation resolved |
| `citations[].control_id` | string | Yes | Which control's narrative this citation appeared in |
| `unresolved_list` | array[object] | Yes | Convenience list of only unresolved citations |

**Citation parsing logic:**
1. Regex pattern: `\[([^\]]+?)\s—\s([^\]]+?)\]`
2. Group 1 = `file_path_extracted`
3. Group 2 = description (logged but not validated)
4. Look up `file_path_extracted` in `registry.file_path_index`
5. If found → resolved. If not found → unresolved (hallucinated).

---

## 3. Output Schemas (Final Deliverables)

These are the two Markdown files written to disk by `write_output.py`.

### 3.1 `rcsa_control_narratives.md`

Structure defined by F5's data model. F6 populates the template.

```markdown
---
title: "RCSA Control Narrative Assessment"
generated: "2026-04-12T14:30:00Z"
controls_assessed: 4
controls_passing: 2
controls_low_confidence: 1
controls_gap: 1
framework: "Custom Demo Controls"
---

## Summary

| Control ID | Control Name | Confidence | Artifacts | Tests | Status |
|---|---|---|---|---|---|
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

[LLM-generated narrative with inline citations in [file_path — description] format]

---

[Repeats for each control]
```

### 3.2 `validation_report.md`

Structure defined by F5's data model. F6 populates from CitationValidationResult.

```markdown
# Validation Report

**Generated**: 2026-04-12T14:30:00Z
**Controls Assessed**: 4

## Citation Resolution

| Citation | Artifact ID | File Path | Resolved | Control |
|---|---|---|---|---|
| auth/oauth_config.yaml — OAuth2 provider... | ART-001 | auth/oauth_config.yaml | ✅ Yes | AC |

**Total Citations**: 8
**Resolved**: 8
**Unresolved**: 0
**Resolution Rate**: 100%

## Evidence Coverage

| Control ID | Artifacts | Tests | Evidence Types Covered | Evidence Types Missing |
|---|---|---|---|---|
| AC | 5 | 3 | auth_config, rbac_policy, session_management | (none) |
| CM | 6 | 3 | change_approval_record, deployment_config, rollback_procedure | (none) |
| DQ | 2 | 1 | input_validation_rule | data_transform_test |
| IH | 2 | 1 | monitoring_config | incident_response_plan, postmortem_record |

## Warnings

- DQ: Missing evidence type `data_transform_test` — no data transformation tests found
- IH: Missing evidence types `incident_response_plan`, `postmortem_record` — no incident response or postmortem documentation found
```

---

## 4. Schema Versioning

All schemas are implicitly `v1` for the demo. Post-demo:
- Add `schema_version: "1.0"` to the root of each data structure
- `validate_input.py` checks schema version before processing
- F5 SKILL.md specifies which schema versions it supports

---

*This document defines the data contracts for F6. All external input schemas are owned by F4. All internal schemas are owned by F6. Output document structure is defined by F5 and populated by F6. Changes to any schema require review by the owners of both sides of the interface.*
