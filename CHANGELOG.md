# Changelog

All notable changes to the SMIF specification are documented here.

Format: `VERSION | DATE | SUMMARY`

Breaking changes are marked **[BREAKING]**.

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
- Appendix C: 52 validation rules — 42 MUST (V-001 through V-042) and 10 SHOULD (W-001 through W-010) — across document, corrections, and workspace scopes
- `examples/financial_risk/` — first complete worked example

---

*This changelog follows the conventions established in CONTRIBUTING.md. Breaking changes require a minimum 14-day comment period before merging.*
