# Semantic Model Interchange Format (SMIF)

**Status:** `0.1.0-draft` — published for early feedback, not yet ratified  
**License:** Apache 2.0

---

SMIF is an open format for representing the semantic layer of a relational database in a form that LLM agents can consume directly. It defines models, columns, relationships, concepts, and metrics — together with provenance, confidence scores, and human correction overlays — as structured YAML that any tool can write and any agent can read.

---

## The problem it solves

LLM agents that reason over databases fail in predictable ways: they misidentify join paths, misinterpret ambiguous column names, and produce syntactically valid SQL that is semantically wrong. The root cause is the absence of a structured semantic layer that tells an agent what data *means*, not just what it *contains*.

Existing semantic layer formats (dbt MetricFlow, Cube, LookML, Wren MDL) require substantial upfront manual modeling effort. SMIF is designed to be generated automatically from schema introspection, sample data, and query logs — then refined by humans through a corrections overlay.

---

## The three-file system

A SMIF workspace contains three files:

| File | Who writes it | Purpose |
|---|---|---|
| `strata.md` | Human (optional, before inference) | Business context and known gotchas. The inference pipeline reads this as prior knowledge. |
| `semantic.yaml` | Tool-generated | The full inferred semantic model. Overwritten on re-inference. |
| `corrections.yaml` | Human (after inference) | Targeted overrides. Never overwritten by tools. Always takes precedence over `semantic.yaml` at serve time. |

---

## Quick example

```yaml
# semantic.yaml (excerpt)
smif_version: "0.1.0"
domain:
  name: "Risk Management Platform"
  description: >
    Internal risk database tracking trader positions, instrument
    valuations, desk-level limits, and daily P&L.

models:
  - model_id: "positions"
    label: "Trader Positions"
    description: >
      Daily snapshot of each trader's open positions. One row per
      trader-instrument-business_date. Primary fact table for
      exposure and P&L analysis.
    physical_source:
      schema: "risk"
      table: "positions"
    columns:
      - name: "net_exposure_usd"
        role: measure
        description: >
          Net exposure in USD. Long = positive, short = negative.
          Post-netting within the same instrument on the same desk.
        provenance:
          source_type: llm_inferred
          confidence: 0.68
          human_reviewed: false
```

A complete example for a financial risk management database is in [`examples/financial_risk/`](examples/financial_risk/).

---

## Reference implementation

[Strata](https://github.com/strata-spec/openstrata) — zero-config Go CLI.
Connects to a PostgreSQL database, infers a SMIF semantic model automatically,
and serves it via MCP.
```bash
go install github.com/strata-spec/openstrata@latest
```

---

## Read the spec

The full specification is in [`SPEC.md`](SPEC.md). It covers:

- The three-file system and pipeline position of each file
- Full schema reference for all object types
- Provenance model and trust hierarchy
- Confidence scoring and schema drift detection
- Corrections overlay with full precedence rules
- Normative validation rules in Appendix C

---

## Required Filters Migration Pattern

Use object entries for `required_filters` on both models and metrics:

- New canonical shape:
  - `expression`: SQL WHERE-clause condition fragment
  - `reason`: plain-language explanation of why omission causes incorrect results

Legacy string-only `required_filters` entries should be migrated to object entries so each filter carries explicit machine-usable logic and human-readable rationale. This guidance is intentionally version-agnostic and applies whenever required filters are present.

---

## Examples

| Example | Domain | What it demonstrates |
|---|---|---|
| [`examples/financial_risk/`](examples/financial_risk/) | Financial risk management | Concepts layer, multi-clause joins, metric required_filters, correction overlay |

More examples are welcome. See `CONTRIBUTING.md`.

---

## JSON Schema

A JSON Schema (draft 2020-12) for machine validation is at:

```
https://strata-spec.github.io/spec/schema/0.1.0/semantic.schema.json
```

The schema skeleton is in [`schema/0.1.0/semantic.schema.json`](schema/0.1.0/semantic.schema.json). It will be completed at v0.1.0 ratification.

---

## Contributing

See [`CONTRIBUTING.md`](CONTRIBUTING.md). The fastest contribution is a new example for a domain you know.

Spec changes require an issue before a PR. The gate for any field addition is the field inclusion test: **does this field change what SQL an agent generates?** If not, it doesn't belong in the spec.

---

## Governance

See [`GOVERNANCE.md`](GOVERNANCE.md). Single maintainer for v0.x. Foundation governance at v1.0 if adoption warrants it.
