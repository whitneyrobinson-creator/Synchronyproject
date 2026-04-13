# F2 Research — What We Learned During Planning

**Feature:** F2 — Orchestration Scripts (Data Dictionary)
**Project:** Synchrony Documentation Automation Skillset
**Last Updated:** April 10, 2026

---

## How F2 Works

F2 is 6 Python scripts that form a pipeline. Raw schema goes in, data dictionary and QA report come out.

### The Pipeline

| Step | Script | What It Does |
|------|--------|--------------|
| 1 | `validate_input.py` | Checks the uploaded schema is valid JSON with the right structure (`table_name`, `source_file`, `fields`). Rejects bad files early. |
| 2 | `extract_fields.py` | Pulls fields out of the schema. Creates a metadata record for each one (name, type, nullable, constraints, etc.). Logs warnings for edge cases like duplicates. |
| 3 | `attach_citations.py` | Merges the LLM's descriptions with the extracted metadata. Validates the LLM's work. Fills placeholders if the LLM failed. Logs any corrections it makes. |
| 4 | `add_timestamps.py` | Adds a `last_verified` timestamp to every field. |
| 5 | `assemble_output.py` | Combines timestamped fields + F3 template → final `data_dictionary.md`. |
| 6 | `generate_qa_report.py` | Produces `qa_report.md` with coverage stats, flagged fields, corrections, and warnings. |

### The File Chain

Every script reads from and writes to `output/intermediate/`. All intermediate files are JSON so you can open them and see exactly what happened at each step.

| File | Created By | Read By |
|------|-----------|---------|
| `user_input.json` | User (via Claude upload) | `validate_input.py` |
| `validated_schema.json` | `validate_input.py` | `extract_fields.py` |
| `extracted_fields.json` | `extract_fields.py` | SKILL.md (F1), `attach_citations.py` |
| `extraction_warnings.json` | `extract_fields.py` (only if warnings exist) | `generate_qa_report.py` |
| `llm_output.json` | F1 (LLM step — Claude writes this file during Step 3) | `attach_citations.py` |
| `merged_fields.json` | `attach_citations.py` | `add_timestamps.py` |
| `timestamped_fields.json` | `add_timestamps.py` | `assemble_output.py`, `generate_qa_report.py` |
| `data_dictionary.md` | `assemble_output.py` | User (final deliverable) |
| `qa_report.md` | `generate_qa_report.py` | User (final deliverable) |

8 files always produced. 1 conditional (`extraction_warnings.json`). 9 max.

---

## Key Decisions and Why

### Python
The Constitution (Section 2) says the team knows Python best. Claude's sandbox runs Python. All three team members are comfortable with it. This was never a debate.

### File-Based (JSON on Disk)
The Constitution (Section 2) requires the team to be able to inspect every intermediate step. JSON files on disk = open in a text editor and see what happened. No database, no in-memory state, no cloud services.

### 6 Separate Scripts (Not 1 Big File)
The Constitution (Principle 3) and feature spec (Section 3) say each script does one job. Beyond that:
- **Easier to debug.** Merge step broken? Open `attach_citations.py`. That's the only place the bug can be.
- **Easier to collaborate.** Whitney works on one script, Sheila works on another. No merge conflicts.
- **Cleaner graceful degradation.** Each script checks its own inputs independently.

### Standard Library Only
No `pip install` in the pipeline. If a third-party package fails to install in Claude's sandbox on demo day, the whole thing breaks. We don't need anything beyond `json`, `os`, `datetime`, and `pathlib`, so why risk it? This was a precautionary call — we didn't test whether `pip install` works in the sandbox. We just decided not to find out on demo day.

### SKILL.md Handles Orchestration
We considered a 7th script (`run_pipeline.py`) that calls the other 6 in order. We didn't build it because SKILL.md already does this job, and the LLM step sits in the middle of the pipeline — a Python script can't call the LLM. SKILL.md can. If the skill ever runs outside Claude as a standalone CLI tool, a runner script would make sense then.

---

## What We Rejected

| Approach | Why Not |
|----------|---------|
| **One big script** | Hard to debug, hard to collaborate on, violates Constitution Principle 3 |
| **Runner script (`run_pipeline.py`)** | SKILL.md already orchestrates; LLM step in the middle makes a Python runner awkward |
| **CLI frameworks (Click, Typer)** | Third-party = install risk in sandbox. `argparse` could work but adds complexity for no benefit — file paths are hardcoded for demo day |
| **Other languages (JS, Go, Bash)** | Constitution says Python. Sandbox runs Python. Team knows Python. |

---

## How LLM Output Validation Works

`attach_citations.py` runs three checks before merging:

1. **Count check** — Did the LLM return the same number of fields we sent?
2. **Name matching** — Does each LLM response match an input field by name? (Case-insensitive.)
3. **Position fallback** — If there are duplicate field names, match by position instead: 1st response → 1st field, 2nd → 2nd, etc.

These are defensive checks. We assumed the LLM could skip fields, return wrong names, or reorder output, and built checks for each case.

---

## Corrections

When `attach_citations.py` fixes something, it logs the correction in the field's `corrections` array. Nothing is silently fixed.

| Type | What Triggered It | What F2 Does |
|------|-------------------|--------------|
| `clarification_flag_override` | Confidence is `"Low"` but flag is `false` | Flips flag to `true` |
| `name_case_mismatch` | LLM returned wrong casing (e.g., `education` vs `EDUCATION`) | Matches case-insensitively, keeps original casing |
| `description_over_limit` | Description over 25 words | Keeps it (soft limit), logs for awareness |
| `order_mismatch` | LLM returned fields in different order | Matches by name, not position |
| `duplicate_name_in_output` | LLM returned two responses for same field | Uses first match, ignores second |

The QA report surfaces all corrections in a "Merge Corrections" section so the team can see what was fixed.

---

## Placeholders

When the LLM fails to describe a field, F2 fills these values:

| Field | Placeholder Value | Type |
|-------|------------------|------|
| `description` | `"[LLM did not return a description]"` | string |
| `confidence` | `"N/A"` | string |
| `evidence_refs` | `["No evidence available — LLM did not process this field"]` | array (single item) |
| `clarification_flag` | `true` | boolean |
| `merge_status` | `"placeholder"` | string |
| `corrections` | `[]` | array (empty) |

**Important:** Types stay consistent between normal and placeholder values. `evidence_refs` is always an array. `confidence` is always a string. Downstream scripts never need to check "is this a placeholder or a real value?" — they process everything the same way.

---

## F1 ↔ F2 Boundary

**F1 controls the LLM. F2 controls everything else.**

- F2 never calls the LLM directly.
- F2 validates whatever file it's given. Good output → merge. Bad or missing output → placeholders.
- F2 owns retry logic — max 2 retries (3 total attempts). If all attempts fail, all fields get placeholders and the pipeline continues.

**One shared field:** `clarification_flag`. The LLM sets it initially. F2 can override `false` → `true` (never the reverse). This is the only field where both features have a say.

---

## Error Handling

### Two Failure Scenarios, Same Outcome

Whether the LLM returns bad output after all retry attempts or never responds at all, the result is the same: all fields get placeholders, and the pipeline keeps running.

### How the Team Knows Something Went Wrong

1. **Console message** — Printed immediately. Shows attempt count, failure reasons, and what action was taken.
2. **QA Report (Processing Notes section)** — Permanent record. The console message disappears when you close the terminal; the QA report stays.
3. **Every field's `merge_status`** — All set to `"placeholder"`. Self-documenting.

### What Does NOT Happen

- Pipeline does not stop with no output (Constitution says always produce something)
- No separate error log file (QA report already covers this)
- No email or notification (file-based only, no cloud services)
- No crash or unhandled exception (this is a known scenario, not an unexpected error)

---

## Testing — What Success Looks Like

- **Determinism:** Same input 3 times → same output except timestamps.
- **Graceful degradation:** Skip the LLM step, run the rest. You still get both deliverables with placeholders and a QA report showing 0% coverage.
- **Field count accuracy:** 25 fields in → 25 field records at every stage.
- **Validation catches bad input:** Feed broken schemas → pipeline rejects them with clear error messages.

---

## Summary

The Constitution and sandbox constraints made most of these decisions for us.

| Decision | Why |
|----------|-----|
| 6 single-purpose scripts | Constitution Principle 3, Feature Spec Section 3 |
| JSON on disk | Constitution Section 2 (team debugging) |
| Standard library only | Sandbox safety, Constitution Principle 3 |
| SKILL.md orchestrates | F1 plan, Claude Agent Skills architecture |
| Python | Constitution Section 2, sandbox requirement |
| Graceful degradation | Constitution Principle 2, Feature Spec US-6 |
| Corrections always logged | Audit trail requirement, QA report design |
