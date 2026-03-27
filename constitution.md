# Project Constitution — Audit-Ready Documentation Generator

> **Purpose:** This constitution defines the principles, boundaries, guardrails, and governance rules for our AI-assisted system that generates audit-ready documentation (data dictionaries, RCSA-style control narratives) from repository artifacts for Synchrony's compliance teams.

---

## Principles

### Principle 1 — Purpose & Scope

This system generates audit-ready documentation — including data dictionaries and RCSA-style control narratives — from repository artifacts. The target users are Synchrony's compliance teams. The system uses AI-assisted development and structured workflows to produce markdown-based documentation outputs.

### Principle 2 — Hallucination Tolerance

- AI must explicitly flag a "gap" if evidence is missing to prevent hallucination.
- ≤ 5% of controls or tests referenced in RCSA narratives are invented (not present in the source artifacts).
- All input validation MUST be handled by script logic before any LLM processing.
- The system MUST validate that every citation resolves to a real artifact.

### Principle 3 — Human-in-the-Loop

The system is designed to not rely too heavily on complex coding by using AI-assisted development and structured workflows. While not everyone on the team is strong in Python, we are still able to troubleshoot issues by using prompt engineering, checking the JSON inputs, and reviewing the markdown outputs. This allows the team to work together to fix problems even if we are not making major changes to the underlying code.

Human review is required on all final output before any report is used or shared. The human can always correct the system even if the "agent" is struggling.

### Principle 4 — Guardrails Are Non-Negotiable

The following rules are hard constraints. If any planned feature or workflow conflicts with a guardrail, the guardrail wins. The conflicting feature must be deferred or redesigned to comply.

Guardrails may be updated, but only through the formal change process defined in the Governance section below — not bypassed quietly.

**Never-Ever Rules:**

- AI must explicitly flag a "gap" if evidence is missing to prevent hallucination.
- ≤ 5% of controls or tests referenced in RCSA narratives are invented (not present in the source artifacts).
- All input validation MUST be handled by script logic before any LLM processing.
- The system MUST validate that every citation resolves to a real artifact.

### Principle 5 — Iterative Refinement

This constitution is versioned and updated as the project learns. It is a living document, not a one-time artifact.

Recurring problems in AI output must trigger a constitution update, not just a one-off workaround. All changes should be documented so the team stays aligned.

---

## Script vs. LLM Ownership

| Responsibility | Owner |
|---|---|
| Input validation | Script logic (before any LLM processing) |
| Citation validation (every citation resolves to a real artifact) | Script logic |
| Prompt construction (task instructions, file structure, markdown template) | Script logic |
| Writing narratives from metadata | LLM (Claude) |
| Flagging gaps when evidence is missing | LLM (Claude) |
| Final review and sign-off | Human (team) |

---

## Security, Privacy & Data Boundaries

### Secrets & Credentials

- Never hardcode API keys, tokens, or passwords in code or configuration files.
- Store all secrets in a local `.env` file.
- The `.env` file must never be committed to version control.
- A `.gitignore` file must be present in the project root and must include `.env` before any version control system is adopted.
- The `.gitignore` file ensures sensitive information stays out of reports.

### What Gets Sent to the LLM

Only the following are sent to Claude's API when a skill runs:

- The prompt (task instructions)
- The specific file structure being analyzed
- The markdown template

### What Is Never Sent

- Raw PII
- API keys or secrets
- Source code containing credentials
- Any data beyond the metadata needed for the LLM to write the narratives

### Data Storage

- The system does not store inputs permanently on Claude.
- Outputs are stored locally as generated markdown documentation files.
- Claude may temporarily store input data for security purposes per Anthropic's policies, but our system does not persist inputs beyond the session.

### Risk Mitigation

If a developer accidentally leaves a "secret" (like a password) in the code, and the AI cites that line of code in a Validation Report, that secret could appear in logs. The fix: the "Never-Ever" rules and the `.gitignore` file ensure sensitive information stays out of reports.

---

## Out of Scope

### Functional Exclusions

The following are out of scope for V1:

- Full documentation generation for entire repositories with many different programming languages, potentially thousands of different files, and potential sensitive information. This is out of scope for the prototype but would be in immediate next steps.
- Documentation for multiple different use cases — we are only showing our demo for our selected public dataset on Kaggle, not data from other use cases from projects 1 and 2. This is out of scope for the demo but would be in the documentation phase (next priority).
- Deployment features — on demo day we are only focusing on demonstrating the documentation generation workflows. Deployment would be in the immediate next steps.
- Automatic repository scanning

### Technical Exclusions

The following architectures, tools, and patterns will not be adopted in V1:

- Microservices
- Cloud deployment
- Frontend UI
- Standalone API
- Real-time processing
- Agent harness libraries (e.g., LangChain, LangGraph, agent-sdk)

---

## Governance

### Change Process

- The whole team needs to agree in order to change the spec.
- Proposed changes are run by our professor to ensure feasibility.
- If there is a new feature proposed, it must be discussed as a team and evaluated based on scope, timeline, and alignment with the constitution.
- Only approved changes are added, and all updates are documented so the team stays aligned.
- Silent workarounds that bypass constitution rules are not permitted. If a rule is not working, amend the constitution — do not ignore it.

### Definition of Done

A task is considered "done" when:

1. The team has all reviewed the work together.
2. It has been run to ensure consistency and accuracy.
3. It passes the evaluation rubric.
4. Every person in the group signs off that they are confident the task is "done."

### Accountability

If someone is not pulling their weight, the team addresses it through the team's established process.

---

> **Version:** v1.0.0 | **Ratified:** 2026-03-27 | **Last Amended:** 2026-03-27
