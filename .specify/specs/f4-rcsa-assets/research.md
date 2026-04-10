# Research: F4 — RCSA Assets

**Feature**: F4 — Templates, Sample Data, and Control Library
**Date**: 2026-04-10
**Status**: Complete — all decisions locked in `plan.md`

---

## Scope of Research

F4 is a static asset package (6 files, zero runtime code). Research focused on four design decisions where multiple valid options existed. No algorithm selection, framework comparison, or architecture trade-offs were required.

---

## Decision 1: Directory Structure

**Question**: How should assets be organized within `skills/rcsa/`?

| Option | Description | Verdict |
|---|---|---|
| **A) Flat directory** — all 6 files in `assets/` | Simple but mixes input files with output templates | ❌ Rejected — unclear which files F6 reads vs. populates |
| **B) Two subdirectories** — `templates/` + `sample-data/` | Separates output-defining files from input-defining files | ✅ Selected |
| **C) Three subdirectories** — `templates/` + `sample-data/` + `config/` | Isolates control library as "config" rather than "sample data" | ❌ Rejected — overengineers for 6 files. Revisit post-handoff if control library needs promotion |

**Rationale**: Option B makes the input/output distinction obvious at the filesystem level. The control library is grouped with sample data because all three JSON files share the same consumption pattern (F6 reads them as inputs), even though the control library is more persistent than the other two. This is documented as a known design decision in `plan.md` (R-08).

---

## Decision 2: Citation Format

**Question**: What syntax should citations use in generated narratives?

| Option | Example | Verdict |
|---|---|---|
| **A) Bracket-index** | `[AC-1]`, `[CM-3]` | ✅ Selected |
| **B) Footnote-style** | `¹`, `²` with endnotes | ❌ Rejected — loses control context; auditor can't tell which control a citation belongs to at a glance |
| **C) Inline path** | `(see auth/oauth_config.yaml:1-42)` | ❌ Rejected — clutters narrative text; paths may change |

**Rationale**: Bracket-index embeds the control ID directly in the citation tag, making it self-documenting. An auditor reading `[AC-1]` immediately knows it's the first evidence citation for Access Control. The tag resolves to a file path + line range via the Citation Index in the validation report.

---

## Decision 3: Sample Data Coverage Distribution

**Question**: Should all four controls have full evidence coverage, or should some have intentional gaps?

| Option | Description | Verdict |
|---|---|---|
| **A) Full coverage for all controls** | Every evidence type has matching sample data | ❌ Rejected — doesn't test gap detection, which is a core feature of the pipeline |
| **B) Mixed coverage with intentional gaps** | AC and CM fully covered; DQ partially covered; IH mostly missing | ✅ Selected |
| **C) Minimal coverage** | Only 1–2 controls have any evidence | ❌ Rejected — doesn't exercise the "full coverage" path; too sparse for a meaningful demo |

**Rationale**: Option B exercises both the "evidence found" and "gap flagged" paths in the pipeline. The specific distribution (AC: 3/3, CM: 3/3, DQ: 1/2, IH: 1/3) creates a realistic spread that demonstrates HIGH, MEDIUM, and LOW confidence tiers in the demo output.

---

## Decision 4: `controls_relevant` in Test Catalog

**Question**: Should the test catalog include a field mapping tests to controls, or should F6 infer this at runtime?

| Option | Description | Verdict |
|---|---|---|
| **A) Include `controls_relevant` field** | Self-documenting; F6 can use as hint or override | ✅ Selected |
| **B) Omit — let F6 infer from file paths/content** | Keeps sample data minimal; avoids potential mismatch | ❌ Rejected — makes sample data opaque without running the pipeline |

**Rationale**: Including `controls_relevant` makes the test catalog independently understandable. A reader can see which controls a test relates to without running F6's evidence mapper. F6 may validate or override this mapping at runtime — the field is a hint, not a constraint.

---

## Open Questions

All research questions were resolved during the plan interview. No open items remain.
