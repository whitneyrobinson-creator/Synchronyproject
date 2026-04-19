# Data Dictionary Skill — Test Results

**Project:** Synchrony Documentation Automation Skillset
**Skill:** Data Dictionary Generation (Skill 1)
**Test Date:** April 18, 2026
**Tester:** Whitney Robinson, Sheila Green, Molly Lowell

---

## Overview

This document records all tests run against the Data Dictionary skill. Tests are organized into three categories: Bad Input Tests (what happens when someone gives the skill broken or incomplete data), Edge Case Tests (what happens with unusual but valid data), and a Generic Dataset Test (proving the skill works beyond the UCI Credit Card dataset).

---

## Bad Input Tests

These tests verify that when someone gives the skill incorrect or broken input, the skill always returns a clear error message — never an empty response or a crash. This is critical because silent failures or empty outputs are unacceptable in an audit context.

| Test | Input | What We're Testing | Why It Matters | Benefit | Actual Output | Pass/Fail |
|---|---|---|---|---|---|---|
| **B1** | JSON file with no table name | What happens when the user forgets to include the name of the database table | Without a table name the skill has no idea what it's documenting. We need it to tell the user clearly what's missing rather than producing a blank or broken output | User gets a specific, actionable error message and knows exactly what to fix before trying again | "table_name is required and must be a non-empty string" — pipeline stopped immediately | ✅ PASS |
| **B2** | JSON file with no source file name | What happens when the user forgets to include where the schema came from | The source file name is how every field gets traced back to its origin. Without it the audit trail breaks. The skill must catch this before anything runs | User is told immediately which required field is missing so they can fix it in one pass | "source_file is required and must be a non-empty string" — pipeline stopped immediately | ✅ PASS |
| **B3** | JSON file with an empty list of fields | What happens when the user uploads a schema that has no fields in it | A schema with no fields has nothing to document. The skill should recognize this and tell the user rather than running a pipeline that produces nothing | User gets a clear message that their schema has no extractable fields instead of an empty data dictionary | "fields is required and must be a non-empty array" — pipeline stopped immediately | ✅ PASS |
| **B4** | A field object that has no field name | What happens when one of the fields in the schema is missing its name | Every field must have a name — it's the most basic piece of information the skill needs. If a field has no name the skill can't identify it, cite it, or describe it | User is told exactly which position in the schema has the problem so they can find and fix it quickly | "fields[0].field_name is required and must be a non-empty string" — pipeline stopped immediately | ✅ PASS |
| **B5** | A JSON file with a syntax error (trailing comma) | What happens when the uploaded file is not valid JSON at all | If the file can't be read as JSON the entire pipeline would fail silently without this check. The skill needs to catch this immediately and explain what went wrong and why | User gets the exact line and column where the error occurred plus a plain-English hint about common causes like trailing commas | "Invalid JSON at line 8, column 5. Cause: trailing comma after type: INTEGER" — pipeline stopped before any processing | ✅ PASS |
| **B6** | A JSON file missing table name, source file, and field names in two fields | What happens when there are multiple problems at once — does the skill report all of them or just the first one | If the skill stops at the first error, the user fixes it, runs again, finds the second error, fixes it, and so on. This wastes time. All problems should be surfaced in one go | User gets a complete list of everything wrong in a single response so they can fix everything at once before re-running | All 4 errors reported simultaneously — missing table_name, missing source_file, fields[0] missing field_name, fields[1] missing field_name | ✅ PASS |

---

## Edge Case Tests

These tests verify that the skill handles unusual but valid inputs correctly — without crashing, skipping fields, or producing misleading output.

| Test | Input | What We're Testing | Why It Matters | Benefit | Actual Output | Pass/Fail |
|---|---|---|---|---|---|---|
| **E1** | A schema with one well-documented field and one field with no metadata at all | What happens when some fields have rich information and others have almost nothing | Real-world schemas are never perfect — some fields are well-documented and some are not. The skill needs to handle both in the same run without breaking | Well-documented fields get accurate High confidence descriptions. Sparse fields get Low confidence and are flagged for human review. No field is skipped or lost | ID = High confidence, MYSTERY_FIELD = Low confidence with clarification flag, 100% coverage, no crash | ✅ PASS |
| **E2** | A schema where two fields have the exact same name | What happens when a schema has duplicate field names — does the skill crash or handle it gracefully | Duplicate field names can happen in real schemas, especially when merging data from multiple sources. The skill needs to process both without losing one or crashing | Both fields are processed and appear in the data dictionary. A warning is added to the QA report so the team knows about the duplicate and can investigate | Both STATUS fields processed independently, Low confidence on both, duplicate warning appeared in QA report Section 5 | ✅ PASS |
| **E3** | A schema where every field has an opaque, meaningless name and no supporting metadata | What happens when there are no strong signals anywhere — does the skill inflate confidence or stay honest | A skill that assigns High confidence to everything regardless of evidence is useless for auditing. The skill must be honest about what it doesn't know | Every field correctly gets Low confidence and is flagged for human review. The QA report shows 0 High, 0 Medium, 3 Low so the team knows the entire schema needs manual review | All 3 fields returned Low confidence, all flagged, QA report showed 0 High 0 Medium 3 Low — note: stale warning from previous test appeared (known limitation — intermediate files persist between runs) | ✅ PASS ⚠️ |
| **E4** | A field whose name contains dots (default.payment.next.month) | What happens when a field name contains special characters like dots — does the skill split it or handle it as one name | Field names with dots are common in certain database systems. If the skill splits the name on dots it would misidentify the field and break the audit trail | The dotted field name is treated as a single complete name throughout the entire pipeline — in the data dictionary, evidence citations, and source path | Field name echoed back exactly as default.payment.next.month, High confidence correctly assigned, source path correct, no splitting or breaking | ✅ PASS |
| **E5** | Running the pipeline with the LLM step deliberately skipped | What happens when the AI component is unavailable — does the whole skill fail or does it still produce something useful | In a real deployment the AI could be offline for maintenance, rate limited, or unavailable. The skill must never give the user nothing — it should always produce something | Both the data dictionary and QA report are still produced with all raw metadata intact. Every field is flagged for manual review. The QA report clearly states the AI was unavailable and shows 0% coverage so the team knows exactly what happened and why | Both deliverables produced, all 25 fields present with placeholder descriptions, 0% coverage, QA report showed LLM availability: unavailable_or_invalid, 3 retries were attempted before falling back to placeholders | ✅ PASS |

---

## Generic Dataset Test

This test proves the skill is not hardcoded to the UCI Credit Card dataset and works on any financial schema.

| Test | Input | What We're Testing | Why It Matters | Benefit | Actual Output | Pass/Fail |
|---|---|---|---|---|---|---|
| **G1** | A 20-field bank loan applications schema (completely different domain from credit cards) | Does the skill work on a schema it has never seen before, in a different financial domain | If the skill only worked on the UCI dataset it would be a demo toy, not a real product. Synchrony needs it to work on their actual internal schemas | The skill processes any properly formatted JSON schema regardless of what the data is about — no configuration changes needed | Full data dictionary and QA report produced, 13 High confidence, 0 Medium, 7 Low, 100% coverage, dotted field name handled correctly, opaque field col_z9 correctly flagged as Low | ✅ PASS |

---

## Known Limitations

| Limitation | Description | Impact | Mitigation |
|---|---|---|---|
| Stale intermediate files | When tests are run back to back without clearing the intermediate folder, warnings from one run can carry over into the next run's QA report | A warning may appear in the QA report that doesn't belong to the current run | Clear the output/intermediate/ folder before each new pipeline run. A future version could add a --clean flag to do this automatically |

---

## Summary

| Category | Tests Run | Tests Passed | Tests Failed |
|---|---|---|---|
| Bad Input Tests | 6 | 6 | 0 |
| Edge Case Tests | 5 | 5 | 0 |
| Generic Dataset Test | 1 | 1 | 0 |
| **Total** | **12** | **12** | **0** |

**All 12 tests passed.** The skill correctly handles bad inputs with clear error messages, processes edge cases without crashing, degrades gracefully when the AI is unavailable, and works on any financial schema — not just the UCI Credit Card dataset.
