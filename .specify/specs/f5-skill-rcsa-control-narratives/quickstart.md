# F5 — SKILL.md: RCSA Control Narrative Generation — Quickstart

**Feature Branch**: `f5-rcsa-skill`
**Created**: 2026-04-10
**Status**: Draft
**Phase**: 1

---

## What This Is

A step-by-step guide to running and testing the F5 SKILL.md. By the end of this doc, you'll have:

- Run the SKILL.md against sample data
- Verified the output matches expected format
- Confirmed confidence scoring and gap detection work correctly

---

## Prerequisites

Before you start:

- [ ] BoodleBox account with access to Claude models
- [ ] The `SKILL.md` file (from `specs/005-skill-rcsa/`)
- [ ] The sample input files (from `specs/005-skill-rcsa/test-data/`)
  - `control_library.yaml`
  - `artifact_registry.yaml`
  - `mapped_evidence.yaml`

---

## Step 1: Set Up the Chat

1. Open BoodleBox and start a new chat
2. Type `@` in the chatbar and select **Claude 4.6 Sonnet** (recommended for iteration) or **Claude 4.6 Opus** (if you need maximum reasoning depth)
3. Attach the `SKILL.md` file using the paperclip icon or drag it into the chat
   - This goes into your Knowledge Bank automatically
   - Optional: Star it (via the brain icon → three-dot menu) so it attaches to every new chat while you're iterating

---

## Step 2: Provide the Sample Input

Paste the following sample input into the chat. This represents what F6 would produce after parsing a repository.

**Prompt:**

```
You are an RCSA assessment assistant. Follow the instructions in the attached SKILL.md exactly.

Here is the input data for this assessment run. Process it according to the workflow defined in the SKILL.md and produce both output documents.

---

CONTROL LIBRARY:

controls:
  - id: "AC-001"
    name: "Access Control"
    objective: "Ensure that access to systems and data is restricted to authorized users based on role and need."
    evidence_types:
      - "authentication_logic"
      - "authorization_checks"
      - "role_definitions"
      - "access_tests"

  - id: "CM-001"
    name: "Change Management"
    objective: "Ensure that changes to production systems follow a documented review and approval process."
    evidence_types:
      - "code_review_config"
      - "branch_protection"
      - "ci_cd_pipeline"
      - "approval_workflows"

  - id: "DQ-001"
    name: "Data Quality"
    objective: "Ensure that data inputs are validated, sanitized, and checked for integrity before processing."
    evidence_types:
      - "input_validation"
      - "schema_validation"
      - "data_integrity_checks"
      - "quality_tests"

  - id: "IH-001"
    name: "Incident Handling"
    objective: "Ensure that security incidents are detected, logged, and escalated through a defined response process."
    evidence_types:
      - "error_handling"
      - "logging_config"
      - "alerting_rules"
      - "incident_response_tests"

---

ARTIFACT REGISTRY:

artifacts:
  - id: "ART-001"
    file_path: "src/auth/login.py"
    artifact_type: "authentication_logic"
    snippet: |
      def authenticate_user(username, password):
          user = user_store.get(username)
          if not user or not verify_hash(password, user.password_hash):
              raise AuthError("Invalid credentials")
          return generate_session_token(user)
    snippet_line_range: "12-17"

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

  - id: "ART-003"
    file_path: "tests/test_auth.py"
    artifact_type: "access_tests"
    snippet: |
      def test_unauthorized_access_denied():
          response = client.get("/admin", headers={})
          assert response.status_code == 401

      def test_role_enforcement():
          viewer = create_user(role="viewer")
          response = client.delete("/records/1", headers=auth(viewer))
          assert response.status_code == 403
    snippet_line_range: "45-53"

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

---

MAPPED EVIDENCE:

mappings:
  - control_id: "AC-001"
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

  - control_id: "CM-001"
    control_name: "Change Management"
    mapped_artifacts:
      - artifact_id: "ART-004"
        artifact_type: "ci_cd_pipeline"
        relevance: "direct"

  - control_id: "DQ-001"
    control_name: "Data Quality"
    mapped_artifacts: []

  - control_id: "IH-001"
    control_name: "Incident Handling"
    mapped_artifacts: []
```

---

## Step 3: Check the Output

The SKILL.md should produce two documents. Here's what to verify for each.

### Output 1: `rcsa_control_narratives.md`

**YAML Front Matter — check these fields exist:**

| Field | Expected Value |
|-------|---------------|
| `title` | "RCSA Control Narrative Assessment" |
| `controls_assessed` | 4 |
| `controls_gap` | 2 |

**Summary Table — check this matches:**

| Control ID | Control Name | Expected Confidence | Expected Gap? |
|------------|-------------|-------------------|---------------|
| AC-001 | Access Control | HIGH | No |
| CM-001 | Change Management | MEDIUM or LOW | No |
| DQ-001 | Data Quality | — | YES |
| IH-001 | Incident Handling | — | YES |

**Per-Control Narratives — check each one:**

| Control | What to Verify |
|---------|---------------|
| AC-001 | 3–5 sentences. Cites all 3 artifacts (ART-001, ART-002, ART-003). Confidence = HIGH. |
| CM-001 | 3–5 sentences. Cites ART-004. Notes that only 1 of 4 evidence types is covered. Confidence = MEDIUM or LOW. |
| DQ-001 | [GAP] flag. States what's missing. Does NOT suggest remediation. Does NOT imply evidence might exist elsewhere. |
| IH-001 | [GAP] flag. States what's missing. Does NOT suggest remediation. Does NOT imply evidence might exist elsewhere. |

### Output 2: `validation_report.md`

| What to Verify | Expected |
|---------------|----------|
| All citations from narratives are listed | Yes |
| Each citation resolves to an artifact ID | Yes |
| Resolution rate | 100% |
| Unresolved citations | 0 |

---

## Step 4: Run the Verification Checklist

Go through each item. If any **Must-Pass** check fails, the SKILL.md needs revision.

### Must-Pass ✅

- [ ] YAML front matter is present and parseable
- [ ] Summary table has exactly 4 rows (one per control)
- [ ] DQ-001 and IH-001 both have `[GAP]` flags
- [ ] AC-001 and CM-001 both have confidence tiers (not GAP)
- [ ] Every citation in the narratives references an artifact from the input
- [ ] No citations reference artifacts that weren't provided
- [ ] GAP narratives do NOT contain compliance claims
- [ ] GAP narratives do NOT suggest remediation steps

### Should-Pass ⚠️

- [ ] Narratives are 3–5 sentences each
- [ ] AC-001 confidence is HIGH (3 direct artifacts)
- [ ] CM-001 confidence is MEDIUM or LOW (1 direct artifact)
- [ ] Citations use the format `[file_path — description]`
- [ ] Summary table confidence values match the per-control sections
- [ ] Validation report lists all citations with resolution status

### Nice-to-Have 💡

- [ ] Narratives read naturally (not robotic or templated)
- [ ] Evidence types covered are listed per control
- [ ] CM-001 narrative notes that only 1 of 4 evidence types was found

---

## Step 5: Iterate

If checks fail:

1. **Identify which check failed** — note the specific control and issue
2. **Check the SKILL.md instructions** — is the relevant rule clear enough?
3. **Revise the SKILL.md** — tighten the instruction, add a worked example, or add an explicit constraint
4. **Re-run** — paste the same sample input and verify the fix

**Common issues and fixes:**

| Issue | Likely Cause | Fix |
|-------|-------------|-----|
| GAP controls get a narrative instead of a flag | SKILL.md doesn't explicitly say "skip narrative for GAP controls" | Add explicit instruction: "If mapped_artifacts is empty, write a GAP statement only" |
| Citations reference artifacts not in the input | SKILL.md doesn't constrain citations to the registry | Add constraint: "Only cite artifacts present in the artifact registry" |
| Confidence is always HIGH | Rubric isn't specific enough | Add the edge case table from the data model |
| GAP narratives suggest remediation | SKILL.md doesn't prohibit it | Add explicit rule: "Do not suggest remediation steps" |
| Narratives are too long | No length constraint | Add: "Each narrative must be 3–5 sentences" |
| Summary table doesn't match narratives | LLM built the table before writing narratives | Reorder instructions: narratives first, then summary table |

---

## Red Team Tests (Optional)

Once the basic run passes, try these adversarial inputs to verify graceful degradation.

### Test: All Controls GAP

Remove all artifacts from the mapped evidence (set all `mapped_artifacts` to `[]`). Expected: 4 GAP flags, no compliance narratives, no hallucinated evidence.

### Test: Snippet Contradicts Metadata

Change ART-001's `artifact_type` to `"logging_config"` but keep the authentication snippet. Expected: LLM trusts the snippet, notes the mismatch, lowers confidence.

### Test: Single Artifact Mapped to Multiple Controls

Add ART-004 to AC-001's mapped artifacts as well as CM-001's. Expected: ART-004 cited in both narratives, overlap noted in each.

---

## Tips

- **Use Claude 4.6 Sonnet for iteration** — it's fast and handles this well. Switch to Opus only if narrative quality or rubric adherence is off.
- **Star the SKILL.md in your Knowledge Bank** while iterating — it auto-attaches to every new chat so you don't have to re-upload each time.
- **Name your chats** (e.g., "F5 Run 3 — fixed GAP handling") so you can track iterations.
- **Move test chats to a folder** (e.g., "F5 Testing") to keep them organized.
- **Use GroupChat** if you want Whitney or Sheila to review a run — share the chat link and they can see the full input/output.

---

*This quickstart covers standalone SKILL.md testing. End-to-end testing with F6 scripts will be documented separately once F6 is built.*
