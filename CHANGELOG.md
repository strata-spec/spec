# Changelog

All notable changes to the SMIF specification are documented here.

Format: `VERSION | DATE | SUMMARY`

Breaking changes are marked **[BREAKING]**.

---

## 0.1.1 | 2026-03 | Additive fields from evaluation findings

- `model.required_filters`: list of `{expression, reason}` objects. Mandatory filters agents must include in every query against a model.
- `metric.required_filters`: upgraded from raw strings to `{expression, reason}` objects. **[BREAKING for metrics — existing raw string values must be migrated]**
- `column.valid_values`: exhaustive list of valid string values for closed-set columns.
- `column.case_sensitive`: boolean indicating whether `valid_values` comparisons are case-sensitive. Defaults to `true` when absent.
- `V-043` added to Appendix C.
- `W-011` added to Appendix C.

---

## 0.1.0-draft | 2026-03 | Initial draft

First public draft of the Semantic Model Interchange Format specification.

Covers:
- The three-file system: `strata.md`, `semantic.yaml`, `corrections.yaml`
- Core object tier: `domain`, `models`, `columns`, `relationships`, `metrics`
- Extended object tier: `concepts`, `query_templates`
- Full provenance model with seven `source_type` values and trust hierarchy
- Confidence scoring with drift-based decay via `ddl_fingerprint`
- Corrections overlay with three `source` values (`user`, `system`, `llm_suggested`) and four `status` values (`approved`, `pending`, `rejected`, `auto_applied`)
- Suppression cascade rules
- Correction provenance contract including the human-in-the-loop guarantee for `llm_suggested` corrections
- Appendix C: 54 validation rules — 43 MUST (V-001 through V-043) and 11 SHOULD (W-001 through W-011) — across document, corrections, and workspace scopes
- `examples/financial_risk/` — first complete worked example

---

*This changelog follows the conventions established in CONTRIBUTING.md. Breaking changes require a minimum 14-day comment period before merging.*
