# Semantic Model Interchange Format (SMIF)

**Specification Version:** 0.1.0-draft  
**Status:** Draft — not yet ratified  
**License:** Apache License 2.0  
**Repository:** https://github.com/strata-spec/spec  
**Maintainers:** Open — see GOVERNANCE.md  
**Last revised:** 2026-03

---

## Abstract

The Semantic Model Interchange Format (SMIF) is an open, version-controlled YAML/JSON schema for representing the semantic layer of a relational database in a form that is directly consumable by LLM agents, text-to-SQL pipelines, and agentic analytics tools. It defines a structured representation of database entities, relationships, concepts, metrics, and usage context — together with provenance, confidence, and human-correction overlays — without requiring a data transformation framework, a BI tool, or manual modeling effort.

This document specifies the format. Tools that generate, validate, or serve SMIF documents are reference implementations of this specification, not the specification itself.

---

## Table of Contents

1. Motivation
2. Design Principles
3. The Three-File System
4. Schema Reference — Document Envelope
5. Schema Reference — Object Tiers and Definitions
   - 5.0 Core and Extended Object Tiers
   - 5.1 The Concepts Layer (Callout)
   - 5.2 `domain` *(core)*
   - 5.3 `models` *(core)*
   - 5.4 `columns` *(core)*
   - 5.5 `relationships` *(core)*
   - 5.6 `metrics` *(core)*
   - 5.7 `concepts` *(extended)*
   - 5.8 `query_templates` *(extended)*
6. Provenance Model
7. Confidence Scoring
8. Corrections Overlay
   - 8.1 `corrections.yaml` structure
   - 8.2 Field definitions
   - 8.3 Valid `correction_type` values
     - 8.3.1 Suppression and cascade rules
   - 8.4 The precedence rule
   - 8.5 Correction provenance contract
9. Example Implementations
10. Explicit Non-Goals
11. Appendix A — JSON Schema (Machine-Readable)
12. Appendix B — Changelog

---

## 1. Motivation

### 1.1 The problem

LLM agents that reason over relational databases fail in predictable ways: they misidentify join paths, misinterpret ambiguously named columns, guess at metric definitions, and produce syntactically valid SQL that is semantically wrong. The root cause is not model capability — it is the absence of a structured, machine-readable semantic layer that tells an agent what the data means, not just what it contains.

Mature solutions for defining semantic layers exist (dbt Semantic Layer / MetricFlow, Cube, LookML, Wren MDL). They work. They also require substantial upfront modeling effort: humans must write YAML, TypeScript, or a domain-specific language against a known schema, maintain it as the schema evolves, and integrate it with a specific transformation or serving framework. For teams without an existing dbt project or Cube deployment, the cost of entry is prohibitive.

At the same time, recent work demonstrates that automatically generating semantic context from schema introspection, sample data, and query logs materially improves text-to-SQL accuracy — by up to 16 percentage points on real-world ambiguous schemas (MotherDuck, 2025) and 8 percentage points in enterprise deployments (Lloyds Banking Group, 2024). The signal exists. The missing piece is a format that captures it in a way that any tool can write and any agent can consume.

### 1.2 The gap this format fills

SMIF is designed for a specific, currently unserved scenario:

> An agent has a connection string to a PostgreSQL database it has never seen before. Within minutes, it should have enough structured semantic context to generate correct SQL for business questions — without a human writing modeling code.

This requires a format that:

- Can be produced by automated inference (schema introspection + LLM annotation + query log mining)
- Can be edited and corrected by humans without special tooling
- Encodes provenance and confidence so agents know how much to trust each field
- Captures human corrections as first-class, versioned, non-destructive overlays
- Is small enough to be transmitted per-query without hitting token limits
- Is agnostic to the tool that generated it and the agent that consumes it

No existing format satisfies all of these constraints. dbt MetricFlow YAML is tied to the dbt graph. Wren MDL is Wren-specific and manually authored. DataHub and OpenMetadata graph models are designed for governance at scale, not per-query context injection. LookML is proprietary.

### 1.3 What SMIF is not

SMIF is not a query engine. It does not execute SQL.  
SMIF is not a data catalog. It does not manage lineage at scale.  
SMIF is not a transformation framework. It does not define dbt models or Spark jobs.  
SMIF is not a governance platform. It does not enforce access control.  
SMIF is not an ontology. It does not support formal reasoning or inheritance chains.

The complete list of non-goals is in Section 10.

### 1.4 The format is the product

The strategic intent of this specification is that SMIF becomes the interchange standard for semantic layers in LLM-agent pipelines — the OpenAPI of database semantics for agents. Tools that generate SMIF documents, serve them over MCP, or consume them in text-to-SQL pipelines are reference implementations of this standard. The standard itself is independent of any of those tools.

---

## 2. Design Principles

### 2.1 The field inclusion test

**A field belongs in SMIF if and only if its presence or value changes what SQL an agent generates, or changes whether an agent should generate SQL at all.**

Fields that exist only for human readability, governance dashboards, or audit logging do not belong in the core schema. They may appear in the `properties` escape hatch (see Section 5.2) but are not part of the spec.

This test is applied to every proposed field addition. It is the primary mechanism for keeping the format small.

### 2.2 Human intent always wins over inference

Semantics produced by automated inference (LLM annotation, query log mining, schema introspection heuristics) are suggestions. Semantics produced by a human (via `corrections.yaml` or `strata.md`) are authoritative. Automated processes MUST NOT overwrite fields with `provenance.source_type: user_defined`. This rule is stated once here and enforced in Section 8.

### 2.3 Provenance is mandatory, not optional

Every substantive field at the object level (model, column, relationship, metric) carries a `provenance` block. There is no such thing as an "unknown source" in a valid SMIF document. If a value was produced by an LLM, that is recorded. If it was inferred from a foreign key constraint, that is recorded. If a human typed it, that is recorded.

This is not metadata for metadata's sake. Agents that know a column description was LLM-inferred from a sparse schema should treat it differently from one that was human-reviewed. The provenance model exists to support that behaviour.

### 2.4 Human-editable and machine-consumable simultaneously

`semantic.yaml` must be readable and editable by a competent engineer without documentation. Field names are self-describing. Structures are flat where possible. The canonical format is YAML for human editing; a semantically equivalent JSON representation is defined for machine consumption. Both are valid SMIF documents.

### 2.5 Confidence decays with schema drift

When the physical schema underlying a model changes (detected via `ddl_fingerprint`), confidence scores on affected objects are decayed automatically by any conforming tool. A SMIF document where fingerprints do not match the live schema MUST be treated as stale by any consuming agent or serving layer.

### 2.6 Extensibility without breaking changes

The `properties` field on any object type provides an escape hatch for tool-specific or domain-specific metadata. Additions to this field never constitute a breaking change. Additions to the core schema increment the minor version. Removals or type changes to existing core fields increment the major version.

---

## 3. The Three-File System

A SMIF-compatible workflow uses three files with distinct roles, ownership, and pipeline positions:

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                   │
│   strata.md          ──► INFERENCE PIPELINE ──►  semantic.yaml   │
│   (human, before)                                (tool-generated) │
│                                                       │           │
│                                               corrections.yaml    │
│                                               (human, after)      │
│                                                       │           │
│                                               ◄── SERVE LAYER ◄── │
│                                               (merged at serve)   │
└─────────────────────────────────────────────────────────────────┘
```

### 3.1 `strata.md` — human intent before inference

`strata.md` is a human-authored Markdown file that the inference pipeline reads as prior knowledge before it inspects the database. It captures intent that cannot be inferred from schema or data: what this database is for, which concepts matter, what the gotchas are, which columns are sensitive.

**When to write it:** Before running `init` for the first time, or before re-running inference on a schema where you know the automated results will be poor.

**When to skip it:** `strata.md` is optional. The pipeline runs without it. Results will be weaker on ambiguous schemas.

**What it is not:** It is not a schema definition. It is not a configuration file. It is plain prose that a human would write to onboard a new colleague onto this database. The inference pipeline uses an LLM to extract structured intent from it; no special syntax is required beyond the light template below.

**What the pipeline does with it:** The pipeline reads `strata.md` and uses its content to condition LLM prompts during the coarse pass (database and table descriptions). It does not parse it programmatically. The content influences, but does not directly write, any field in `semantic.yaml`.

**Light template:**

```markdown
# Database: <name>

## What this database is for
<One paragraph describing the business domain and primary use case.>

## Key business concepts
<List the 3-10 concepts that matter most. These become candidates for the `concepts` layer.>
- <ConceptName>: <one-sentence definition>

## Known gotchas
<Anything that would trip up someone writing SQL for the first time.>
- <Example: "customer_id in the orders table refers to the account, not the individual contact">

## Sensitive columns
<List columns that contain PII or confidential data. The pipeline will not sample these.>
- <schema.table.column>

## Do not infer metrics from
<Tables where query log patterns should be ignored — staging tables, audit logs, etc.>
- <schema.table>
```

The sections are labelled for human clarity; the pipeline reads all of them. Sections may be omitted or extended freely.

### 3.2 `semantic.yaml` — the generated semantic model

`semantic.yaml` is the primary output of the inference pipeline. It is the canonical SMIF document. Its schema is fully defined in Sections 4 and 5.

**Who writes it:** The inference tool writes it. Humans may edit it directly, but should prefer `corrections.yaml` for targeted changes, since `semantic.yaml` is overwritten on re-inference.

**When it is regenerated:** On every `init` run. Existing `corrections.yaml` entries are preserved and reapplied after regeneration.

**What it contains:** The full inferred semantic model — domain metadata, models, columns, relationships, concepts, metrics, query templates, and provenance for all of the above.

### 3.3 `corrections.yaml` — the human override layer

`corrections.yaml` is a human-authored overlay that takes unconditional precedence over `semantic.yaml` at serve time. It contains a list of correction records, each of which targets a specific object and field in `semantic.yaml` and replaces or augments it.

**Who writes it:** Humans, directly or via a tool that exposes a `correct` command. It may also receive agent-suggested corrections (with `source: llm_suggested`) pending human approval.

**When it is applied:** At serve time, the serving layer merges `corrections.yaml` on top of `semantic.yaml` before exposing any data to agents. Corrections are never written into `semantic.yaml`.

**Precedence rule (stated once, enforced everywhere):**

> Fields in `corrections.yaml` with `source: user_defined` or `source: system` and `status: approved` ALWAYS override the corresponding field in `semantic.yaml`. Automated processes — including the inference pipeline, any downstream tool, and any LLM agent — MUST NOT modify or delete entries in `corrections.yaml` with `source: user_defined`.

The full `corrections.yaml` schema is defined in Section 8.

### 3.4 File co-location

All three files are designed to live together in a single directory, version-controlled alongside any application or pipeline code:

```
myproject/
  strata.md           # Optional. Human-authored prior knowledge.
  semantic.yaml       # Tool-generated. Re-created on re-inference.
  corrections.yaml    # Human-authored overlay. Never overwritten by tools.
```

`semantic.json` is an optional machine-readable export of `semantic.yaml` in JSON format, with identical content and no additional fields. Tools may generate it alongside `semantic.yaml`. It is not the canonical format.

---

## 4. Schema Reference — Document Envelope

Every valid `semantic.yaml` document begins with the following top-level fields:

```yaml
smif_version: "0.1.0"
generated_at: "2026-03-20T14:32:00Z"
tool_version: "mysmiftool/0.1.0"
source:
  type: postgres
  host_fingerprint: "sha256:a3f..."    # SHA-256 of host:port:dbname, not the DSN
  schema_names:
    - public
    - risk
domain: { ... }
models: [ ... ]
relationships: [ ... ]
concepts: [ ... ]
metrics: [ ... ]
query_templates: [ ... ]
```

| Field | Type | Required | Description |
|---|---|---|---|
| `smif_version` | string | Yes | Semver string matching the spec version this document conforms to. |
| `generated_at` | string (ISO 8601) | Yes | UTC timestamp of document generation. |
| `tool_version` | string | Yes | Identifying string for the tool and version that generated this document. |
| `source.type` | string | Yes | Database type. `postgres` is the only defined value in v0.1. |
| `source.host_fingerprint` | string | Yes | Non-reversible fingerprint of the connection target. Used to detect when `semantic.yaml` is being served against a different database than the one it was generated from. MUST NOT contain the DSN, password, or hostname in plaintext. |
| `source.schema_names` | list of strings | Yes | The Postgres schema names that were introspected. |
| `domain` | object | Yes | See Section 5.1. |
| `models` | list of objects | Yes | See Section 5.2. Must contain at least one entry. |
| `relationships` | list of objects | No | See Section 5.4. |
| `concepts` | list of objects | No | See Section 5.5. |
| `metrics` | list of objects | No | See Section 5.6. |
| `query_templates` | list of objects | No | See Section 5.7. |

---

## 5. Schema Reference — Object Tiers and Definitions

### 5.0 Core and Extended Object Tiers

SMIF objects are divided into two conformance tiers.

**Core objects** — a SMIF-compliant tool MUST implement these. A document missing any core object type is invalid (except where the object list is optional, as noted in Section 4).

| Object type | Required in document |
|---|---|
| `domain` | Yes — exactly one |
| `models` | Yes — at least one |
| `columns` | Yes — at least one per model |
| `relationships` | No — list may be empty |
| `metrics` | No — list may be empty |
| `corrections` | No — `corrections.yaml` may be absent |

**Extended objects** — a SMIF-compliant tool is NOT required to generate or serve these. A tool that does generate or serve them MUST conform to the schemas in Sections 5.7 and 5.8. Consuming agents MUST handle their absence gracefully — they are never a prerequisite for generating SQL.

| Object type | Required in document |
|---|---|
| `concepts` | No |
| `query_templates` | No |

The two-tier distinction also provides a migration path: tools implement core first, extensions when ready. A v0.1-core-only implementation is fully conformant.

---

> ### 5.1 What is the concepts layer, and why does it exist?
>
> Most semantic layer formats represent the physical database closely: a model maps to a table, a column maps to a column, a metric maps to an aggregation. This works well for BI queries where users know they are asking about tables.
>
> LLM agents do not think in tables. They think in business concepts: "revenue", "active customer", "open position", "net exposure". A user asking "what is our net exposure to European equities?" is not asking about a table called `risk_position` with a column called `calc_net_exp_usd`. They are expressing a concept — and the agent needs a bridge between the concept and the physical schema.
>
> The `concepts` layer is that bridge. It defines named business concepts in human language, maps them to the models and columns that express them, and optionally provides a SQL definition that the agent can use directly. Concepts are not metrics — a concept can be a business entity ("Trader"), a classification ("Equity Instrument"), or a calculation ("Net Exposure"). Metrics are a subset of concepts that involve aggregation and are given their own top-level type for that reason.
>
> The concepts layer is optional. A SMIF document without it is valid. But agents operating over domains with rich business vocabulary — finance, healthcare, logistics — will generate substantially better SQL when concepts are defined.

---

### 5.2 `domain` *(core)*

The `domain` block provides database-level context. It is read by agents at the start of a session to orient the LLM before it inspects individual models.

```yaml
domain:
  name: "Risk Management Platform"
  description: >
    Internal risk management database tracking trader positions, instrument
    valuations, desk-level limits, and daily P&L for the equities division.
  key_concepts:
    - "Position: a trader's current holding in a specific instrument"
    - "Net Exposure: the sum of long and short positions after netting"
    - "Breach: a position or desk limit that has been exceeded"
  temporal_model:
    business_date_column: "business_date"
    timezone: "Europe/London"
    note: "All timestamps are stored in UTC. business_date reflects London close."
  known_gotchas:
    - "instrument_id in positions refers to internal instrument IDs, not ISIN or Bloomberg tickers"
    - "deleted_at IS NULL must always be appended to queries against the instruments table"
    - "P&L figures are in USD regardless of instrument currency"
  provenance:
    source_type: llm_inferred
    confidence: 0.72
    human_reviewed: false
```

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | Yes | Human-readable name for this database or system. Used in agent context headers. |
| `description` | string | Yes | 2–5 sentence description of what this database is for, who uses it, and what domain it covers. This is injected into agent system prompts. |
| `key_concepts` | list of strings | No | Short, plain-language definitions of the 3–10 most important business concepts. Each entry is a string in the form `"ConceptName: definition"`. |
| `temporal_model` | object | No | Describes how time is represented in this database. |
| `temporal_model.business_date_column` | string | No | Canonical column name (unqualified) used to represent business date across tables where it appears. |
| `temporal_model.timezone` | string | No | IANA timezone string for the primary business timezone. |
| `temporal_model.note` | string | No | Free-text note on timestamp conventions, e.g. storage vs display timezone behaviour. |
| `known_gotchas` | list of strings | No | Plain-language descriptions of non-obvious schema behaviours that cause SQL errors. Each entry should be a complete sentence. These are injected verbatim into agent context at query time. |
| `provenance` | object | Yes | See Section 6. |

---

### 5.3 `models` *(core)*

A `model` is the SMIF representation of a queryable entity — typically a table or view, but potentially a named SQL expression. Each model has a name, a physical source, a set of columns, and optional keys.

```yaml
models:
  - model_id: "positions"
    name: "positions"
    label: "Trader Positions"
    description: >
      Daily snapshot of each trader's open positions. One row per
      trader-instrument-business_date combination. Use this table as the
      primary fact table for exposure and P&L analysis.
    physical_source:
      schema: "risk"
      table: "positions"
    primary_key: "position_id"
    ddl_fingerprint: "sha256:7c4e..."
    row_count_estimate: 4200000
    columns: [ ... ]   # see Section 5.3
    provenance:
      source_type: schema_constraint
      confidence: 1.0
      human_reviewed: false
    properties: {}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `model_id` | string | Yes | Stable identifier for this model. Used in relationships, concepts, metrics, and corrections. Must be unique within the document. Lowercase, underscores, no spaces. |
| `name` | string | Yes | The physical table or view name. |
| `label` | string | Yes | Human-readable display name. Used in agent responses and UI. |
| `description` | string | Yes | 1–3 sentences describing what this model represents, its grain, and its primary use. This is the single most important field for agent accuracy on ambiguous schemas. |
| `physical_source` | object | Yes | Identifies the physical object in the database. |
| `physical_source.schema` | string | Yes | Postgres schema name (e.g. `public`, `risk`). |
| `physical_source.table` | string | Yes | Table or view name. |
| `physical_source.sql` | string | No | If the model is defined by a SQL expression rather than a table, this field contains that expression. `physical_source.table` should not be set if `sql` is set. |
| `primary_key` | string or list of strings | No | Column name or names constituting the primary key. Derived from schema constraints where available. Used for relationship resolution. |
| `ddl_fingerprint` | string | Yes | SHA-256 hash of the normalised DDL for this table at the time of generation. Used to detect schema drift. See Section 7.3. |
| `row_count_estimate` | integer | No | Approximate row count at time of generation. Used by agents for query planning heuristics (e.g., whether to suggest LIMIT clauses). Not a live value. |
| `columns` | list of objects | Yes | Must contain at least one entry. See Section 5.3. |
| `provenance` | object | Yes | See Section 6. |
| `properties` | object | No | Escape hatch for tool-specific or domain-specific metadata. Any key-value pairs. Not part of the core spec. |

---

### 5.4 `columns` *(core)*

Columns are defined as a list nested under their parent model. Every column in a model that participates in any query — as a filter, join key, projection, grouping, or aggregation — MUST be present. Columns that exist in the physical table but have no semantic meaning (internal system columns, staging artefacts) MAY be omitted; their absence signals to agents that they should not be used.

```yaml
columns:
  - name: "position_id"
    data_type: "bigint"
    role: identifier
    label: "Position ID"
    description: "Surrogate primary key. Has no business meaning. Do not filter or display."
    nullable: false
    cardinality_category: high
    example_values: []
    usage_profile:
      select_frequency: 0.12
      where_frequency: 0.04
      group_by_frequency: 0.0
      join_frequency: 0.31
      common_clauses: ["JOIN positions ON position_id = ..."]
    difficulty: self_evident
    provenance:
      source_type: schema_constraint
      confidence: 1.0
      human_reviewed: false

  - name: "net_exposure_usd"
    data_type: "numeric"
    role: measure
    label: "Net Exposure (USD)"
    description: >
      Calculated net exposure in USD for this position. Long positions are positive,
      short positions are negative. Values are post-netting within the same instrument.
      Do not sum this column across instruments without grouping by instrument_id first.
    nullable: true
    cardinality_category: high
    example_values: ["-4250000.00", "127500.00", "0.00"]
    usage_profile:
      select_frequency: 0.61
      where_frequency: 0.08
      group_by_frequency: 0.0
      join_frequency: 0.0
      common_clauses: ["SUM(net_exposure_usd)", "WHERE net_exposure_usd > 0"]
    difficulty: domain_dependent
    provenance:
      source_type: llm_inferred
      source_ref: "xiyan_pass_2"
      confidence: 0.68
      human_reviewed: false
```

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | Yes | Physical column name exactly as it appears in the database. |
| `data_type` | string | Yes | PostgreSQL data type string (e.g. `bigint`, `text`, `numeric`, `timestamptz`). |
| `role` | enum | Yes | Semantic role. Valid values: `identifier`, `dimension`, `measure`, `timestamp`, `flag`. See Role Definitions below. |
| `label` | string | Yes | Human-readable display name. |
| `description` | string | Yes | Full description of what this column contains. Must include: what the value means, units where applicable, any filters required for correct interpretation, and common misinterpretations. This is the primary field used by agents for schema linking. |
| `nullable` | boolean | Yes | Whether the column allows NULL values. Used by agents when deciding whether to include COALESCE or IS NULL conditions. |
| `cardinality_category` | enum | Yes | Approximate cardinality. Valid values: `low` (< 100 distinct values), `medium` (100–10,000), `high` (> 10,000). Derived from sampling. |
| `example_values` | list of strings | No | 3–10 representative values as strings, sanitised of PII. For `identifier` role columns, this SHOULD be empty. For `dimension` role columns with `low` cardinality, this SHOULD be populated. |
| `usage_profile` | object | No | Statistics derived from query log mining. See Usage Profile below. |
| `difficulty` | enum | No | Annotation difficulty classification. Valid values: `self_evident`, `context_dependent`, `ambiguous`, `domain_dependent`. See Difficulty Taxonomy below. |
| `provenance` | object | Yes | See Section 6. |

#### Role Definitions

| Role | Definition | Agent behaviour |
|---|---|---|
| `identifier` | A surrogate or natural key with no intrinsic business meaning (e.g. `position_id`, `uuid`). | Agents use for joins; do not display or filter unless explicitly requested. |
| `dimension` | A descriptive attribute suitable for grouping, filtering, or display (e.g. `instrument_type`, `desk_name`). | Agents group and filter on these. |
| `measure` | A numeric value representing a quantity, amount, or rate (e.g. `net_exposure_usd`, `trade_count`). | Agents aggregate these; do not group by these. |
| `timestamp` | A date, time, or datetime value representing when something occurred or a business date (e.g. `trade_date`, `created_at`). | Agents use for time-range filters and time-series grouping. |
| `flag` | A boolean or low-cardinality indicator (e.g. `is_active`, `is_deleted`, `status`). | Agents filter on these; do not aggregate. |

#### Difficulty Taxonomy

| Value | Definition | Pipeline action |
|---|---|---|
| `self_evident` | Column name and type are sufficient to infer meaning (e.g. `customer_id`, `created_at`). | Low-confidence LLM description acceptable. |
| `context_dependent` | Meaning requires table context but not domain knowledge (e.g. `status` in a `trades` table). | Include table description in LLM prompt. |
| `ambiguous` | Meaning cannot be reliably inferred even with table context (e.g. `value`, `flag`, `code`). | Flag for human review. Set `human_reviewed: false`, confidence ≤ 0.4. |
| `domain_dependent` | Correct interpretation requires domain knowledge not derivable from schema or logs (e.g. `calc_net_exp_usd` in a risk system). | Flag for human review. Consider populating `strata.md`. |

#### Usage Profile

The `usage_profile` object is populated from query log mining (e.g. `pg_stat_statements`). All frequency values are floats in [0.0, 1.0] representing the fraction of queries in the sample that used this column in that clause type.

| Field | Type | Description |
|---|---|---|
| `select_frequency` | float | Fraction of sampled queries selecting this column. |
| `where_frequency` | float | Fraction of sampled queries filtering on this column. |
| `group_by_frequency` | float | Fraction of sampled queries grouping by this column. |
| `join_frequency` | float | Fraction of sampled queries joining on this column. |
| `common_clauses` | list of strings | 1–5 representative normalised clause fragments (with literals replaced by `?`). Derived from query annotation or pattern mining. |

If `usage_profile` is absent, agents MUST treat it as equivalent to all-zero frequencies.

---

### 5.5 `relationships` *(core)*

Relationships define join paths between models. They are derived from foreign key constraints (preferred), query log join patterns (secondary), or human definition (highest confidence).

```yaml
relationships:
  - relationship_id: "positions_to_instruments"
    from_model: "positions"
    from_column: "instrument_id"
    to_model: "instruments"
    to_column: "instrument_id"
    relationship_type: many_to_one
    join_condition: "positions.instrument_id = instruments.instrument_id"
    semantic_role: "Resolves instrument metadata for a position"
    always_valid: true
    preferred: true
    provenance:
      source_type: schema_constraint
      confidence: 1.0
      human_reviewed: false

  - relationship_id: "positions_to_limits"
    from_model: "positions"
    from_column: "desk_id"
    to_model: "desk_limits"
    to_column: "desk_id"
    relationship_type: many_to_one
    join_condition: >
      positions.desk_id = desk_limits.desk_id
      AND positions.business_date = desk_limits.effective_date
    semantic_role: "Joins position to its applicable limit on the same business date"
    always_valid: false
    preferred: true
    provenance:
      source_type: log_inferred
      source_ref: "queryid:3847291"
      confidence: 0.74
      human_reviewed: false
```

| Field | Type | Required | Description |
|---|---|---|---|
| `relationship_id` | string | Yes | Stable identifier. Unique within the document. |
| `from_model` | string | Yes | `model_id` of the model on the "many" or "left" side. |
| `from_column` | string | Yes | Column name in `from_model` used in the join condition. |
| `to_model` | string | Yes | `model_id` of the model on the "one" or "right" side. |
| `to_column` | string | Yes | Column name in `to_model` used in the join condition. |
| `relationship_type` | enum | Yes | Valid values: `one_to_one`, `one_to_many`, `many_to_one`, `many_to_many`. |
| `join_condition` | string | Yes | Full SQL join condition, fully qualified with model names. May be multi-clause (e.g. date-scoped joins). |
| `semantic_role` | string | No | Plain-language description of what this relationship represents and when to use it. Agents use this to choose between multiple possible join paths to the same target model. |
| `always_valid` | boolean | Yes | If `true`, this join is safe to apply in any query that references both models. If `false`, the join has conditions or caveats described in `semantic_role` that the agent must evaluate. |
| `preferred` | boolean | Yes | If `true` and multiple relationships exist between the same pair of models, this is the default join path. Exactly one relationship between any model pair should be `preferred: true`. |
| `provenance` | object | Yes | See Section 6. |

---

### 5.6 `metrics` *(core)*

A `metric` is a named, aggregated business measure with well-defined semantics: an expression, an aggregation type, optionally a set of valid and invalid dimensions, and any required filters. Metrics are a specialised form of concept. They are separated into their own type because they carry aggregation semantics that affect SQL generation.

```yaml
metrics:
  - metric_id: "total_net_exposure"
    name: "total_net_exposure"
    label: "Total Net Exposure"
    description: >
      Total net exposure in USD across all positions. Sums net_exposure_usd
      across all non-deleted positions for a given business date.
      Always filter by business_date; do not sum across dates without
      explicit user instruction.
    expression: "SUM(positions.net_exposure_usd)"
    aggregation: sum
    status: active
    default_time_dimension:
      model: "positions"
      column: "business_date"
    valid_dimensions:
      - model: "positions"
        column: "desk_id"
      - model: "positions"
        column: "instrument_id"
      - model: "instruments"
        column: "asset_class"
    invalid_dimensions:
      - model: "positions"
        column: "position_id"
        reason: "Surrogate key. Grouping by this produces one row per position, not a meaningful aggregate."
    required_filters:
      - "positions.deleted_at IS NULL"
    additivity: fully_additive
    provenance:
      source_type: log_inferred
      source_ref: "queryid:1029384,queryid:1029392"
      confidence: 0.79
      human_reviewed: false
```

| Field | Type | Required in `semantic.yaml` | Description |
|---|---|---|---|
| `metric_id` | string | Yes | Stable identifier. Unique within the document. |
| `name` | string | Yes | Programmatic name. |
| `label` | string | Yes | Human-readable display name. |
| `description` | string | Yes | Full description including units, required context, and any caveats. |
| `expression` | string | Yes | SQL expression defining the metric. Uses fully qualified `model_name.column_name` references. |
| `aggregation` | enum | Yes | Top-level aggregation function. Valid values: `sum`, `count`, `count_distinct`, `avg`, `min`, `max`, `ratio`, `custom`. |
| `status` | enum | **Runtime only — MUST NOT appear in `semantic.yaml`** | Operational status computed by the serving layer at startup based on correction cascades. Valid values: `active`, `degraded`, `suppressed`. See metric status values below and validation rule V-035. |
| `degraded_reason` | string | **Runtime only — MUST NOT appear in `semantic.yaml`** | Populated by the serving layer when `status: degraded`. Plain-language explanation of which suppressed dependency caused degradation. See validation rule V-036. |
| `default_time_dimension` | object | No | The model and column that should be used for time-series analysis of this metric by default. |
| `valid_dimensions` | list of objects | No | Model–column pairs that are semantically valid GROUP BY dimensions for this metric. Each entry has `model` and `column`. |
| `invalid_dimensions` | list of objects | No | Model–column pairs that are semantically invalid for this metric, with a `reason` string. Agents MUST NOT group by invalid dimensions unless the user explicitly overrides. |
| `required_filters` | list of strings | No | SQL filter conditions that MUST be present in any query using this metric. These protect against common errors (e.g., double-counting deleted rows). |
| `additivity` | enum | No | Indicates whether the metric can be safely summed across all dimensions. Valid values: `fully_additive` (safe to SUM across any dimension), `semi_additive` (safe to SUM across some dimensions, e.g. balance accounts), `non_additive` (ratios, percentages — cannot be summed). |
| `provenance` | object | Yes | See Section 6. |

#### Metric `status` values

These values are assigned by the serving layer at runtime — they are never stored in `semantic.yaml`. The serving layer computes `status` for every metric after applying correction cascades (Section 8.3.1) and exposes it as part of the served representation.

| Value | Definition | Agent behaviour |
|---|---|---|
| `active` | All dependencies are available and no suppression cascade affects this metric. | Use normally. |
| `degraded` | One or more columns or models referenced in the metric's `expression` or `required_filters` have been suppressed. The metric may still be partially usable. | Surface `degraded_reason` to the user before generating SQL. Do not use silently. |
| `suppressed` | The metric itself has been explicitly suppressed via a `suppress` correction, or its parent model has been suppressed. | Exclude from agent context entirely. |

---

### 5.7 `concepts` *(extended)*

A `concept` is a named business entity or classification that spans one or more models and columns. Concepts provide the vocabulary layer that bridges natural language questions to physical schema.

`concepts` is an extended object. A conforming tool is not required to generate or serve concepts. A consuming agent encountering a document without `concepts` MUST NOT fail. The concepts layer materially improves agent accuracy on domain-rich schemas but is never a prerequisite for generating SQL.

```yaml
concepts:
  - concept_id: "net_exposure"
    label: "Net Exposure"
    description: >
      The aggregate value of a trader's or desk's open positions after netting
      long and short positions in the same instrument. Positive values indicate
      net long; negative values indicate net short. Always expressed in USD.
    definition_sql: >
      SELECT desk_id, instrument_id, SUM(net_exposure_usd) AS net_exposure
      FROM risk.positions
      WHERE deleted_at IS NULL
        AND business_date = :business_date
      GROUP BY desk_id, instrument_id
    broader: "exposure"
    maps_to_models:
      - "positions"
    maps_to_columns:
      - model: "positions"
        column: "net_exposure_usd"
    provenance:
      source_type: llm_inferred
      confidence: 0.61
      human_reviewed: false

  - concept_id: "breach"
    label: "Limit Breach"
    description: >
      A condition where a trader's or desk's net exposure in an instrument
      exceeds the approved limit for that desk on that business date.
      A breach exists when net_exposure_usd > desk_limits.limit_usd for
      the matching desk_id and effective_date.
    definition_sql: >
      SELECT p.desk_id, p.instrument_id, p.business_date,
             SUM(p.net_exposure_usd) AS net_exposure,
             dl.limit_usd,
             SUM(p.net_exposure_usd) - dl.limit_usd AS breach_amount
      FROM risk.positions p
      JOIN risk.desk_limits dl
        ON p.desk_id = dl.desk_id
        AND p.business_date = dl.effective_date
      WHERE p.deleted_at IS NULL
      GROUP BY p.desk_id, p.instrument_id, p.business_date, dl.limit_usd
      HAVING SUM(p.net_exposure_usd) > dl.limit_usd
    broader: null
    maps_to_models:
      - "positions"
      - "desk_limits"
    maps_to_columns:
      - model: "positions"
        column: "net_exposure_usd"
      - model: "desk_limits"
        column: "limit_usd"
    provenance:
      source_type: user_defined
      confidence: 1.0
      human_reviewed: true
```

| Field | Type | Required | Description |
|---|---|---|---|
| `concept_id` | string | Yes | Stable identifier. Unique within the document. |
| `label` | string | Yes | Human-readable name for this concept. Should match how business users refer to it. |
| `description` | string | Yes | Full description of what this concept means, how it is calculated, caveats, and units. This is injected directly into agent context when a query appears to reference this concept. |
| `definition_sql` | string | No | A complete, parameterised SQL query that defines this concept. Parameters use `:param_name` syntax. If present, agents MAY use this directly rather than generating SQL from scratch. If absent, agents must synthesise SQL from the description and referenced models/columns. |
| `broader` | string or null | No | `concept_id` of a broader concept that this concept is a sub-type of (e.g. `net_exposure` is a sub-type of `exposure`). Used for disambiguation when multiple concepts could satisfy a query. |
| `maps_to_models` | list of strings | No | `model_id` values that this concept draws from. Agents use this to determine which models to include in queries involving this concept. |
| `maps_to_columns` | list of objects | No | Specific model–column pairs that are central to this concept. Each entry has `model` (a `model_id`) and `column` (a column name). |
| `provenance` | object | Yes | See Section 6. |

---

### 5.8 `query_templates` *(extended)*

A `query_template` is a parameterised SQL pattern that has been validated — either from observed query logs or human authorship — and represents a known-correct way to answer a class of questions. It is the highest-confidence semantic object in a SMIF document: unlike inferred descriptions, a template is observed behaviour, not a prediction.

`query_templates` is an extended object. A conforming tool is not required to generate or serve query templates. A tool that does must conform to the schema below. A consuming agent encountering a document without `query_templates` MUST NOT fail — templates are an acceleration mechanism, not a dependency.

```yaml
query_templates:
  - template_id: "desk_exposure_by_date"
    description: "Net exposure per desk for a given business date, ordered by absolute exposure descending."
    sql_template: >
      SELECT p.desk_id, d.desk_name, SUM(p.net_exposure_usd) AS net_exposure_usd
      FROM risk.positions p
      JOIN risk.desks d ON p.desk_id = d.desk_id
      WHERE p.business_date = :business_date
        AND p.deleted_at IS NULL
      GROUP BY p.desk_id, d.desk_name
      ORDER BY ABS(SUM(p.net_exposure_usd)) DESC
    parameters:
      - name: business_date
        data_type: date
        description: "The business date to report on. Format: YYYY-MM-DD."
    provenance:
      source_type: log_inferred
      source_ref: "queryid:9920011"
      confidence: 0.88
      human_reviewed: false
```

| Field | Type | Required | Description |
|---|---|---|---|
| `template_id` | string | Yes | Stable identifier. Unique within the document. |
| `description` | string | Yes | Plain-language description of what question this template answers. Agents use this for semantic matching. |
| `sql_template` | string | Yes | Complete SQL query with parameters in `:param_name` syntax. Must be valid PostgreSQL. |
| `parameters` | list of objects | No | Each parameter has `name`, `data_type`, and `description`. Agents use `description` to map user-provided values to parameters. |
| `provenance` | object | Yes | See Section 6. |

---

## 6. Provenance Model

Every substantive object in a SMIF document — models, columns, relationships, concepts, metrics, query templates, and the domain block — carries a `provenance` object. Provenance is mandatory. A SMIF document with any substantive object missing a `provenance` block is invalid.

### 6.1 The `provenance` object

```yaml
provenance:
  source_type: log_inferred          # Required. See trust hierarchy below.
  source_ref: "queryid:3847291"      # Optional. Tool-specific reference to the source.
  agent: "smiftool/0.1.0"            # Optional. Tool or human that produced this value.
  timestamp: "2026-03-20T14:32:00Z"  # Optional. When this provenance was recorded.
  confidence: 0.74                   # Required. See Section 7.
  human_reviewed: false              # Required. See Section 7.
```

| Field | Type | Required | Description |
|---|---|---|---|
| `source_type` | enum | Yes | The provenance type. See trust hierarchy below. |
| `source_ref` | string | No | A tool-specific reference to the specific source. For `log_inferred`, this is typically a `queryid` from `pg_stat_statements`. For `code_extracted`, a file path and line number. For `catalog_import`, a catalog entity URN. |
| `agent` | string | No | Identifier for the process or person that produced this value. |
| `timestamp` | string (ISO 8601) | No | When this provenance record was created. |
| `confidence` | float [0.0–1.0] | Yes | See Section 7. |
| `human_reviewed` | boolean | Yes | Whether a human has explicitly reviewed and accepted this value. See Section 7. |

### 6.2 Valid `source_type` values and trust hierarchy

The trust hierarchy is ordered from highest to lowest trust. When a field's value conflicts across sources, the highest-trust source wins.

| Rank | `source_type` | Definition |
|---|---|---|
| 1 | `user_defined` | Set directly by a human in `corrections.yaml` or by editing `semantic.yaml`. Highest trust. MUST NOT be overwritten by any automated process. |
| 2 | `ddl_comment` | Extracted from a `COMMENT ON COLUMN` or `COMMENT ON TABLE` statement in the database DDL. Represents explicit human intent stored in the database. |
| 3 | `schema_constraint` | Derived from schema metadata: primary key constraints, foreign key constraints, NOT NULL constraints, check constraints, unique constraints. |
| 4 | `catalog_import` | Imported from an external metadata catalog (DataHub, OpenMetadata, Atlan). Trust depends on the catalog's own data quality; treat as high-confidence human-authored metadata unless the catalog's provenance indicates otherwise. |
| 5 | `code_extracted` | Extracted from transformation or application code: dbt model definitions, SQLAlchemy ORM models, migration files, application-layer docstrings. |
| 6 | `log_inferred` | Inferred from query log analysis (e.g. `pg_stat_statements`). Represents observed usage, not intended semantics. |
| 7 | `llm_inferred` | Generated by an LLM prompt. Lowest trust among automated sources. |

### 6.3 Provenance conflicts

When the same field receives input from multiple sources (e.g. both a `ddl_comment` and an `llm_inferred` description exist for the same column), the value from the higher-trust source takes precedence. The `provenance` block records the source of the value that was actually used; superseded values are not stored in the core document.

If a tool wishes to retain superseded values for debugging, they MAY be placed in the `properties` escape hatch with a key like `_superseded_provenance`.

---

## 7. Confidence Scoring

### 7.1 Confidence values

The `confidence` field is a float in [0.0, 1.0] representing the tool's estimated probability that this field value is correct and will not mislead an agent.

General thresholds used by conforming tools and agents:

| Range | Meaning | Agent behaviour |
|---|---|---|
| 0.9 – 1.0 | High confidence. Derived from constraints or human review. | Use without qualification. |
| 0.7 – 0.9 | Moderate confidence. LLM-inferred with supporting evidence. | Use, but prefer alternatives if available. |
| 0.5 – 0.7 | Low confidence. Inferred from weak or sparse signal. | Use with caution; surface to user on ambiguous queries. |
| 0.0 – 0.5 | Very low confidence. Minimal evidence; likely ambiguous column. | Flag for human review; do not use in automated decisions without confirmation. |

### 7.2 `human_reviewed`

`human_reviewed: true` means a human has explicitly reviewed the value of this field and either confirmed it, edited it, or added it. It overrides confidence for purposes of agent trust: a `human_reviewed: true` field with `confidence: 0.6` is more trustworthy than a `human_reviewed: false` field with `confidence: 0.9`. Agents SHOULD treat `human_reviewed: true` fields as equivalent to `confidence: 1.0` for planning purposes.

### 7.3 Confidence decay from schema drift

`ddl_fingerprint` on each model stores a SHA-256 hash of the normalised DDL for that table at the time of generation. When a conforming serving tool starts up, it SHOULD re-hash the live DDL for each model and compare against stored fingerprints.

If fingerprints differ:

- The `description` of the model and all its columns MUST have their confidence decayed by a factor of 0.5.
- The `relationships` involving that model MUST have their confidence decayed by a factor of 0.7.
- Metrics referencing that model MUST have their confidence decayed by a factor of 0.7.
- All decayed fields SHOULD have `human_reviewed` set to `false` (even if previously `true`) until re-reviewed.
- The serving tool MUST surface a warning to connected agents that the semantic model is stale for affected models.

Fingerprint drift does not cause the serving tool to refuse to serve. It causes it to communicate the staleness so agents can factor it into their responses.

### 7.4 Confidence assignment during inference

Reference implementations SHOULD assign confidence according to these defaults:

| Inference source | Default confidence |
|---|---|
| `schema_constraint` | 1.0 |
| `ddl_comment` | 0.95 |
| `catalog_import` | 0.85 |
| `code_extracted` | 0.80 |
| `log_inferred` (>= 10 supporting queries) | 0.75 |
| `log_inferred` (< 10 supporting queries) | 0.55 |
| `llm_inferred` (difficulty: self_evident) | 0.80 |
| `llm_inferred` (difficulty: context_dependent) | 0.65 |
| `llm_inferred` (difficulty: ambiguous) | 0.40 |
| `llm_inferred` (difficulty: domain_dependent) | 0.35 |

Tools MAY adjust these defaults based on model-specific self-evaluation prompts or multi-pass agreement.

---

## 8. Corrections Overlay

`corrections.yaml` is the authoritative overlay layer. Its entries take unconditional precedence over the corresponding fields in `semantic.yaml`. The merge is applied at serve time, not at generation time.

The corrections format version is tied to the SMIF spec version. When the spec version increments, the changelog (Appendix B) MUST state whether existing `corrections.yaml` files are forward-compatible with the new version. If migration is required, the spec ships with migration instructions or a migration tool. There is no separate corrections schema version.

### 8.1 `corrections.yaml` structure

```yaml
smif_version: "0.1.0"
corrections:
  - correction_id: "corr_001"
    target_type: column
    target_id: "positions.net_exposure_usd"
    correction_type: description_override
    new_value: >
      Net exposure in USD for this position as of market close. Long = positive,
      short = negative. This value is post-netting within the same instrument
      on the same desk. Never sum across business_date without explicit
      instruction. USD regardless of instrument currency.
    reason: "LLM description omitted the cross-date summing caveat."
    source: user_defined
    status: approved
    author: "alice@example.com"
    timestamp: "2026-03-21T09:14:00Z"

  - correction_id: "corr_002"
    target_type: relationship
    target_id: "positions_to_limits"
    correction_type: join_override
    new_value: >
      positions.desk_id = desk_limits.desk_id
      AND positions.business_date = desk_limits.effective_date
      AND desk_limits.deleted_at IS NULL
    reason: "Log-inferred join was missing the soft-delete filter on desk_limits."
    source: user_defined
    status: approved
    author: "bob@example.com"
    timestamp: "2026-03-22T11:02:00Z"

  - correction_id: "corr_003"
    target_type: metric
    target_id: "total_net_exposure"
    correction_type: required_filter_add
    new_value: "positions.hedge_flag = false"
    reason: "Agent suggested excluding hedged positions from default exposure calculation."
    source: llm_suggested
    status: pending
    session_id: "session_abc123"
    timestamp: "2026-03-23T10:45:00Z"
```

### 8.2 `corrections.yaml` field definitions

| Field | Type | Required | Description |
|---|---|---|---|
| `smif_version` | string | Yes | SMIF spec version this corrections file conforms to. Must match the `smif_version` in the corresponding `semantic.yaml`. |
| `corrections` | list | Yes | List of correction records. |
| `correction_id` | string | Yes | Stable, unique identifier for this correction. Human-friendly (e.g. `corr_001`). |
| `target_type` | enum | Yes | The type of object being corrected. Valid values: `domain`, `model`, `column`, `relationship`, `concept`, `metric`, `query_template`. |
| `target_id` | string | Yes | Reference to the object being corrected. Convention: for columns, `model_id.column_name`. For all other objects, the object's own `*_id` field value. |
| `correction_type` | enum | Yes | The nature of the correction. See Section 8.3. |
| `new_value` | any | Yes | The corrected value. Type must match the field being corrected. |
| `reason` | string | No | Human explanation of why this correction was made. Assists future reviewers. |
| `source` | enum | Yes | Who produced this correction. Valid values: `user_defined`, `system`, `llm_suggested`. See Section 8.5 for full semantics. |
| `status` | enum | Yes | Processing status of this correction. Valid values: `approved`, `pending`, `rejected`. See Section 8.5. |
| `author` | string | No | Identifier of the person or system that created this correction. |
| `session_id` | string | No | For `source: llm_suggested` corrections, the session identifier in which the correction was proposed. Used for review traceability. |
| `timestamp` | string (ISO 8601) | Yes | When this correction was recorded. |

### 8.3 Valid `correction_type` values

| `correction_type` | Applies to | Effect |
|---|---|---|
| `description_override` | model, column, concept, metric, domain | Replaces the `description` field entirely. |
| `label_override` | model, column, concept, metric | Replaces the `label` field. |
| `role_override` | column | Replaces the `role` field. |
| `join_override` | relationship | Replaces the `join_condition` field. |
| `always_valid_override` | relationship | Replaces the `always_valid` field. |
| `preferred_override` | relationship | Replaces the `preferred` field. |
| `required_filter_add` | metric | Appends an entry to `required_filters`. |
| `invalid_dimension_add` | metric | Appends an entry to `invalid_dimensions`. |
| `example_values_override` | column | Replaces `example_values`. |
| `definition_sql_override` | concept, metric | Replaces `definition_sql` or `expression`. |
| `confidence_override` | any | Replaces the `confidence` value on the target object's `provenance`. Use when human review has confirmed or corrected an LLM-inferred value. |
| `suppress` | model, column, relationship, metric | Marks the object as suppressed. See cascade rules in Section 8.3.1. |

#### 8.3.1 Suppression and cascade rules

Suppression cascades downward in the dependency graph. Suppressing a model suppresses everything that depends on it. Suppressing a column does not suppress its parent model — suppression never propagates upward. The following rules are mandatory for conforming serving layers.

**Suppressing a model:**
- The model is excluded from agent context.
- All columns of that model are implicitly suppressed — they are excluded from agent context and MUST NOT appear in schema representations served to agents.
- All relationships where that model appears as `from_model` or `to_model` are excluded from agent context.
- All metrics whose `expression` or `required_filters` reference that model are set to `status: suppressed`.
- All `query_templates` whose `sql_template` references that model are excluded from agent context.

**Suppressing a column:**
- The column is excluded from agent context.
- All relationships where that column appears as `from_column` or `to_column` are excluded from agent context.
- All metrics whose `expression` or `required_filters` directly reference that column are set to `status: degraded`, and the serving layer MUST populate `degraded_reason` with a plain-language explanation identifying the suppressed column. A metric is not suppressed by a single suppressed column — it may still be partially usable with caveats.

**Suppressing a metric:**
- Only that metric is excluded from agent context. No cascade.

**Suppressing a relationship:**
- Only that relationship is excluded from agent context. The models and columns it references remain visible.

The serving layer applies cascade rules at startup and on any `corrections.yaml` change. `status` on metrics is a derived, runtime field — it MUST NOT be set manually in `semantic.yaml` or `corrections.yaml`. Its only valid manipulation is via a `suppress` correction on a dependency, which causes the serving layer to set it automatically.

### 8.4 The precedence rule (restated in full)

1. Corrections with `source: user_defined` or `source: system` and `status: approved` ALWAYS override the corresponding field in `semantic.yaml`.
2. Corrections with `source: llm_suggested` and `status: pending` MUST NOT be applied to agent context. They are stored, surfaced for review, and only applied once a human sets `status: approved`.
3. The inference pipeline (`init`) MUST NOT read `corrections.yaml` and apply corrections back into `semantic.yaml`. The overlay is resolved at serve time only.
4. Re-running `init` MUST NOT remove, modify, or overwrite any entry in `corrections.yaml`.
5. A conforming tool MUST NOT set `source_type: user_defined` on any `provenance` block in `semantic.yaml`. That provenance value is reserved for objects whose authoritative source is a human correction in `corrections.yaml`, and is applied by the serving layer at merge time, not written into `semantic.yaml` by the inference tool.

---

### 8.5 Correction provenance contract

This section defines what each `source` value means procedurally and what a conforming serving layer MUST do with it. The how of review interfaces and agent APIs is left to implementations; the what is specified here.

#### Source: `user_defined`

A human explicitly authored this correction — directly editing `corrections.yaml`, or using a tool command (`correct`) that writes on their behalf.

- `status` MUST be `approved` when written.
- Applied immediately at serve time. No approval gate.
- Highest trust. MUST NOT be overwritten or removed by any automated process.
- Sets effective `provenance.source_type: user_defined` on the target field at merge time.

> **Note on authority vs origin.** The `source` field describes the *origin* of a correction, not the authority level of the person who made it. `source: user_defined` intentionally does not distinguish between user roles (analyst, operator, administrator). The spec has no opinion on organisational trust hierarchies — that is an access control concern, not a semantic format concern. Implementations requiring role-based trust hierarchies (e.g. operator corrections bypass the review gate, analyst corrections do not) should enforce this through their own access control layer rather than extending the `source` enum. Future proposals to add role-based source values (e.g. `admin_defined`) are out of scope for this specification.

#### Source: `system`

Generated by the tool's own automated processes: drift detection (fingerprint mismatch), log mining re-runs, or re-inference on an updated schema.

- Applied automatically only when the target field's existing provenance is `log_inferred` or `llm_inferred`.
- MUST NOT be applied when the target field's existing provenance is `user_defined` (i.e. a prior `source: user_defined` correction exists for the same target). The system correction is stored with `status: rejected` in that case, and `reason` MUST explain the conflict.
- Written with `status: auto_applied` to distinguish from human-approved corrections.

Valid `status` values for `source: system`: `auto_applied`, `rejected`.

#### Source: `llm_suggested`

An agent proposed this correction during an active query session, typically via the `record_correction` MCP tool.

- Written to `corrections.yaml` immediately with `status: pending`.
- MUST NOT be applied to serving context while `status: pending`. A conforming serving layer that applies a `pending` correction to agent context is non-conformant.
- A conforming serving layer MUST expose pending corrections to human reviewers through some mechanism (CLI listing, UI flag, or API endpoint). The mechanism is implementation-defined; the requirement to expose them is not.
- Once a human sets `status: approved`, the correction behaves identically to `source: user_defined`.
- If a human sets `status: rejected`, the correction is retained in `corrections.yaml` for audit purposes but never applied.
- `session_id` SHOULD be populated so reviewers can trace which query session generated the suggestion.

**The single conformance requirement:** a serving layer MUST NOT apply a correction with `source: llm_suggested` and `status: pending` to agent context under any circumstances. This is the human-in-the-loop guarantee. Implementations may build richer review workflows but cannot bypass this gate.

#### `status` field summary

| `status` | Set by | Meaning |
|---|---|---|
| `approved` | Human (always) | Correction is applied at serve time. |
| `pending` | Tool (for `llm_suggested`) | Correction is stored but not applied. Awaiting human review. |
| `rejected` | Human, or tool (for `system` source conflicts) | Correction is retained for audit, never applied. |
| `auto_applied` | Tool (for `system` source only) | Correction was applied automatically by the tool's drift or mining process. |

**Validation constraint:** `status: auto_applied` is only valid when `source: system`. The combination `source: user_defined, status: auto_applied` or `source: llm_suggested, status: auto_applied` is a spec violation. Conforming implementations of the `validate` command MUST reject documents containing this combination and MUST surface it as an error, not a warning.

---

## 9. Example Implementations

The SMIF specification repository includes example implementations in the `examples/` directory. Each example contains a complete set of three files (`strata.md`, `semantic.yaml`, `corrections.yaml`) for a realistic domain, along with annotations explaining provenance decisions and correction rationale.

Current examples:

| Example | Domain | Description |
|---|---|---|
| `examples/financial_risk/` | Financial risk management | Positions, instruments, traders, desks, limits, and P&L. Demonstrates the concepts layer, multi-clause join conditions, and metric required_filters. |
| `examples/ecommerce/` | E-commerce | Orders, customers, products, inventory, and fulfilment. Demonstrates the full MVSM with log-inferred semantics and a correction overlay. |
| `examples/saas_analytics/` | SaaS product analytics | Events, users, sessions, subscriptions, and feature flags. Demonstrates timestamp handling, flag columns, and semi-additive metrics. |

Examples are maintained alongside the specification. A pull request that changes a field definition in the spec MUST include a corresponding update to at least one example.

To add a new example: fork the repository, add a directory under `examples/`, include all three files, and open a pull request with a description of the domain and what it demonstrates.

---

## 10. Explicit Non-Goals

The following capabilities are out of scope for SMIF. They will not be added to the core schema. Tools that need them are welcome to use the `properties` escape hatch.

**Formal ontology reasoning and inheritance chains.** SMIF has a `broader` field on concepts for simple parent-child classification. It does not support OWL-style reasoning, transitivity, or class hierarchies. If you need FIBO alignment or full ontology traversal, use an ontology tool and import the relevant definitions into SMIF concepts via `catalog_import` provenance.

**Cross-database identity resolution.** SMIF describes one database. Joining or reconciling entities across multiple databases, federating queries, or resolving that "customer" in database A is the same as "client" in database B is outside scope.

**Temporal logic and slowly changing dimensions.** SMIF acknowledges that databases contain time-related columns and provides a `temporal_model` block for basic conventions. It does not model SCD2 history tables, bitemporal semantics, or point-in-time correctness. These require runtime query context that cannot be represented in a static semantic model.

**Query optimisation and execution planning.** SMIF is not a query planner. It does not contain index metadata, partition information, or cost statistics. Tools MAY use `row_count_estimate` as a rough planning heuristic, but the format makes no guarantees about query performance.

**Access control enforcement.** SMIF does not define or enforce column-level security, row-level security, or attribute-based access control. Downstream tools are responsible for these. SMIF may record that a column is sensitive (via `difficulty: domain_dependent` or a `properties` flag) to inform agent behaviour, but it does not enforce access.

**BI dashboard semantics.** SMIF does not model drill-through hierarchies, pivot configurations, chart types, or dashboard layouts. It is a semantic layer for agents generating SQL, not a BI metadata layer.

**Full ontology mapping to standards like FIBO or HL7.** Mapping a financial database to FIBO or a clinical database to HL7 is a valuable and non-trivial engineering task. SMIF makes it possible to record the results of such mapping (via the `concepts` layer and `catalog_import` provenance) but does not provide a mapping engine or a canonical set of financial or clinical concepts.

**Multi-tenant or multi-user semantic versioning.** SMIF is a file format, not a collaboration platform. Concurrent editing, merge conflict resolution, and per-user semantic views are not addressed by the format. These are the responsibility of the tooling and version control system the user chooses to manage the files in.

---

## Appendix A — JSON Schema (Machine-Readable)

A JSON Schema (draft 2020-12) for validating SMIF documents is maintained at:

```
https://strata-spec.github.io/spec/schema/0.1.0/semantic.schema.json
```

The schema file is present in this repository at `schema/0.1.0/semantic.schema.json`. The full machine-readable schema will be completed and the URL activated at v0.1.0 ratification. The current file in the repository is a structural skeleton.

The schema enforces:

- Required fields at each object level
- Valid enum values for `role`, `relationship_type`, `aggregation`, `additivity`, `source_type`, `correction_type`, `difficulty`, and `cardinality_category`
- Float ranges for `confidence`
- ISO 8601 format for timestamp fields
- Non-empty strings for required string fields
- `model_id` and `column` cross-reference validity within the document

The schema does NOT enforce:

- Semantic consistency (e.g. that a `measure` column is numeric)
- SQL validity of `expression`, `join_condition`, or `definition_sql` fields
- Correctness of `ddl_fingerprint` against a live database

Validators implementing the `validate` command SHOULD use the JSON Schema for structural validation and MAY add semantic validation rules on top of it.

---

## Appendix B — Changelog

| Version | Date | Summary |
|---|---|---|
| 0.1.0-draft | 2026-03 | Initial draft. Full schema for domain, models, columns, relationships, concepts, metrics, query_templates. Three-file system. Provenance model. Confidence scoring. Corrections overlay. |
| 0.1.0-draft.2 | 2026-03 | **Breaking changes from draft.1:** (1) Two-tier object model introduced — concepts and query_templates demoted to extended tier; (2) `metrics[].status` added (`active` \| `degraded` \| `suppressed`) — runtime-computed, MUST NOT appear in `semantic.yaml`; (3) `metrics[].degraded_reason` added — runtime-computed, MUST NOT appear in `semantic.yaml`; (4) `corrections[].status` added (`approved` \| `pending` \| `rejected` \| `auto_applied`) — required field; (5) `corrections[].session_id` added — optional, for `llm_suggested` traceability; (6) `corrections[].source` enum rationalised: `admin_defined` removed, `system` added; valid values are now `user_defined`, `system`, `llm_suggested`; (7) `smif_corrections_version` removed — corrections version is now tied to `smif_version`; (8) Suppression cascade rules formalised in Section 8.3.1; (9) Correction provenance contract added as Section 8.5. |


---

## Appendix C — Validation Rules (Normative)

This appendix is normative. The rules defined here carry the same weight as the schema definitions in Sections 4–8. The `validate` command in any conforming SMIF implementation is a direct runner for these test cases and nothing more. A tool claiming SMIF conformance MUST pass all MUST-tier rules against the canonical test cases in this appendix. SHOULD-tier rules are advisory for conformance purposes.

### C.1 Rule tiers

| Tier | Identifier prefix | Validator exit code | Output channel |
|---|---|---|---|
| MUST | `V-` | Non-zero (file is not valid SMIF) | stderr |
| SHOULD | `W-` | Zero (file is valid SMIF with warnings) | stderr |

### C.2 Rule scopes

| Scope | Label | Files required |
|---|---|---|
| `document` | Validated against a single file in isolation | `semantic.yaml` only, or `corrections.yaml` only |
| `corrections` | Validated against `corrections.yaml` in isolation | `corrections.yaml` only |
| `workspace` | Requires both files present | `semantic.yaml` + `corrections.yaml` |

---

### C.3 MUST rules — `V-` series

---

#### V-001 — `smif_version` present and non-empty
**Scope:** document  
**Description:** The top-level `smif_version` field MUST be present and MUST be a non-empty string conforming to semver (MAJOR.MINOR.PATCH or MAJOR.MINOR.PATCH-label).

**Passes:**
```yaml
smif_version: "0.1.0"
```

**Fails:**
```yaml
# smif_version absent entirely
generated_at: "2026-03-20T14:32:00Z"
```

---

#### V-002 — `generated_at` is valid ISO 8601 UTC
**Scope:** document  
**Description:** The `generated_at` field MUST be present and MUST be a valid ISO 8601 datetime string in UTC (Z suffix or +00:00 offset).

**Passes:**
```yaml
generated_at: "2026-03-20T14:32:00Z"
```

**Fails:**
```yaml
generated_at: "20 March 2026"
```

---

#### V-003 — At least one model present
**Scope:** document  
**Description:** The `models` list MUST be present and MUST contain at least one entry.

**Passes:**
```yaml
models:
  - model_id: "orders"
    name: "orders"
    # ...
```

**Fails:**
```yaml
models: []
```

---

#### V-004 — Each model has at least one column
**Scope:** document  
**Description:** Every model object MUST contain a `columns` list with at least one entry.

**Passes:**
```yaml
models:
  - model_id: "orders"
    columns:
      - name: "order_id"
        # ...
```

**Fails:**
```yaml
models:
  - model_id: "orders"
    columns: []
```

---

#### V-005 — `source.type` is a defined value
**Scope:** document  
**Description:** `source.type` MUST be one of the defined values. In v0.1.0 the only defined value is `postgres`.

**Passes:**
```yaml
source:
  type: postgres
```

**Fails:**
```yaml
source:
  type: mysql
```

---

#### V-006 — `model_id` unique within document
**Scope:** document  
**Description:** No two models in the `models` list MUST share the same `model_id`.

**Passes:**
```yaml
models:
  - model_id: "orders"
  - model_id: "customers"
```

**Fails:**
```yaml
models:
  - model_id: "orders"
  - model_id: "orders"
```

---

#### V-007 — `relationship_id` unique within document
**Scope:** document  
**Description:** No two entries in the `relationships` list MUST share the same `relationship_id`.

**Passes:**
```yaml
relationships:
  - relationship_id: "orders_to_customers"
  - relationship_id: "orders_to_products"
```

**Fails:**
```yaml
relationships:
  - relationship_id: "orders_to_customers"
  - relationship_id: "orders_to_customers"
```

---

#### V-008 — `metric_id` unique within document
**Scope:** document  
**Description:** No two entries in the `metrics` list MUST share the same `metric_id`.

**Passes:**
```yaml
metrics:
  - metric_id: "total_revenue"
  - metric_id: "order_count"
```

**Fails:**
```yaml
metrics:
  - metric_id: "total_revenue"
  - metric_id: "total_revenue"
```

---

#### V-009 — `concept_id` unique within document
**Scope:** document  
**Description:** No two entries in the `concepts` list MUST share the same `concept_id`.

**Passes:**
```yaml
concepts:
  - concept_id: "active_customer"
  - concept_id: "churned_customer"
```

**Fails:**
```yaml
concepts:
  - concept_id: "active_customer"
  - concept_id: "active_customer"
```

---

#### V-010 — `template_id` unique within document
**Scope:** document  
**Description:** No two entries in the `query_templates` list MUST share the same `template_id`.

**Passes:**
```yaml
query_templates:
  - template_id: "revenue_by_month"
  - template_id: "top_customers"
```

**Fails:**
```yaml
query_templates:
  - template_id: "revenue_by_month"
  - template_id: "revenue_by_month"
```

---

#### V-011 — Provenance block present on all substantive objects
**Scope:** document  
**Description:** Every entry in `models`, `relationships`, `metrics`, `concepts`, `query_templates`, and the `domain` block MUST contain a `provenance` object. Columns nested within models MUST also carry `provenance`.

**Passes:**
```yaml
models:
  - model_id: "orders"
    columns:
      - name: "order_id"
        provenance:
          source_type: schema_constraint
          confidence: 1.0
          human_reviewed: false
    provenance:
      source_type: schema_constraint
      confidence: 1.0
      human_reviewed: false
```

**Fails:**
```yaml
models:
  - model_id: "orders"
    columns:
      - name: "order_id"
    # provenance absent
```

---

#### V-012 — `confidence` in range [0.0, 1.0]
**Scope:** document  
**Description:** Every `provenance.confidence` value MUST be a float in the closed interval [0.0, 1.0].

**Passes:**
```yaml
provenance:
  source_type: llm_inferred
  confidence: 0.72
  human_reviewed: false
```

**Fails:**
```yaml
provenance:
  source_type: llm_inferred
  confidence: 1.4
  human_reviewed: false
```

---

#### V-013 — `human_reviewed` is boolean
**Scope:** document  
**Description:** Every `provenance.human_reviewed` value MUST be a boolean (`true` or `false`), not a string or null.

**Passes:**
```yaml
provenance:
  source_type: schema_constraint
  confidence: 1.0
  human_reviewed: false
```

**Fails:**
```yaml
provenance:
  source_type: schema_constraint
  confidence: 1.0
  human_reviewed: "no"
```

---

#### V-014 — `provenance.source_type` is a defined value
**Scope:** document  
**Description:** Every `provenance.source_type` MUST be one of the seven defined values: `user_defined`, `ddl_comment`, `schema_constraint`, `catalog_import`, `code_extracted`, `log_inferred`, `llm_inferred`.

**Passes:**
```yaml
provenance:
  source_type: log_inferred
  confidence: 0.74
  human_reviewed: false
```

**Fails:**
```yaml
provenance:
  source_type: human_guessed
  confidence: 0.5
  human_reviewed: false
```

---

#### V-015 — `column.role` is a defined value
**Scope:** document  
**Description:** Every column `role` field MUST be one of the five defined values: `identifier`, `dimension`, `measure`, `timestamp`, `flag`.

**Passes:**
```yaml
columns:
  - name: "created_at"
    role: timestamp
```

**Fails:**
```yaml
columns:
  - name: "created_at"
    role: date
```

---

#### V-016 — `relationship_type` is a defined value
**Scope:** document  
**Description:** Every `relationship_type` field MUST be one of: `one_to_one`, `one_to_many`, `many_to_one`, `many_to_many`.

**Passes:**
```yaml
relationships:
  - relationship_id: "orders_to_customers"
    relationship_type: many_to_one
```

**Fails:**
```yaml
relationships:
  - relationship_id: "orders_to_customers"
    relationship_type: belongs_to
```

---

#### V-017 — `metric.aggregation` is a defined value
**Scope:** document  
**Description:** Every metric `aggregation` field MUST be one of: `sum`, `count`, `count_distinct`, `avg`, `min`, `max`, `ratio`, `custom`.

**Passes:**
```yaml
metrics:
  - metric_id: "order_count"
    aggregation: count
```

**Fails:**
```yaml
metrics:
  - metric_id: "order_count"
    aggregation: COUNT
```

---

#### V-018 — `metric.additivity` is a defined value when present
**Scope:** document  
**Description:** When `additivity` is present on a metric, it MUST be one of: `fully_additive`, `semi_additive`, `non_additive`.

**Passes:**
```yaml
metrics:
  - metric_id: "conversion_rate"
    additivity: non_additive
```

**Fails:**
```yaml
metrics:
  - metric_id: "conversion_rate"
    additivity: not_additive
```

---

#### V-019 — `column.cardinality_category` is a defined value
**Scope:** document  
**Description:** Every column `cardinality_category` field MUST be one of: `low`, `medium`, `high`.

**Passes:**
```yaml
columns:
  - name: "status"
    cardinality_category: low
```

**Fails:**
```yaml
columns:
  - name: "status"
    cardinality_category: sparse
```

---

#### V-020 — `column.difficulty` is a defined value when present
**Scope:** document  
**Description:** When `difficulty` is present on a column, it MUST be one of: `self_evident`, `context_dependent`, `ambiguous`, `domain_dependent`.

**Passes:**
```yaml
columns:
  - name: "calc_net_exp_usd"
    difficulty: domain_dependent
```

**Fails:**
```yaml
columns:
  - name: "calc_net_exp_usd"
    difficulty: hard
```

---

#### V-021 — At most one `preferred: true` relationship per model pair
**Scope:** document  
**Description:** For any pair of models (A, B), there MUST NOT be more than one relationship entry where `preferred: true`. Model pairs are unordered for this purpose — a relationship from A to B and a relationship from B to A are considered the same pair.

**Passes:**
```yaml
relationships:
  - relationship_id: "orders_to_customers_primary"
    from_model: "orders"
    to_model: "customers"
    preferred: true
  - relationship_id: "orders_to_customers_historical"
    from_model: "orders"
    to_model: "customers"
    preferred: false
```

**Fails:**
```yaml
relationships:
  - relationship_id: "orders_to_customers_primary"
    from_model: "orders"
    to_model: "customers"
    preferred: true
  - relationship_id: "orders_to_customers_alt"
    from_model: "orders"
    to_model: "customers"
    preferred: true
```

---

#### V-022 — Relationship `from_model` resolves to a defined model
**Scope:** document  
**Description:** Every `relationship.from_model` value MUST match a `model_id` present in the `models` list.

**Passes:**
```yaml
models:
  - model_id: "orders"
relationships:
  - relationship_id: "orders_to_customers"
    from_model: "orders"
```

**Fails:**
```yaml
models:
  - model_id: "orders"
relationships:
  - relationship_id: "r1"
    from_model: "invoices"   # no model with this id
```

---

#### V-023 — Relationship `to_model` resolves to a defined model
**Scope:** document  
**Description:** Every `relationship.to_model` value MUST match a `model_id` present in the `models` list.

**Passes:**
```yaml
models:
  - model_id: "customers"
relationships:
  - relationship_id: "orders_to_customers"
    to_model: "customers"
```

**Fails:**
```yaml
models:
  - model_id: "customers"
relationships:
  - relationship_id: "r1"
    to_model: "accounts"   # no model with this id
```

---

#### V-024 — Relationship `from_column` resolves to a column in `from_model`
**Scope:** document  
**Description:** Every `relationship.from_column` value MUST match the `name` of a column in the model identified by `from_model`.

**Passes:**
```yaml
models:
  - model_id: "orders"
    columns:
      - name: "customer_id"
relationships:
  - from_model: "orders"
    from_column: "customer_id"
```

**Fails:**
```yaml
models:
  - model_id: "orders"
    columns:
      - name: "customer_id"
relationships:
  - from_model: "orders"
    from_column: "client_id"   # not a column in orders
```

---

#### V-025 — Relationship `to_column` resolves to a column in `to_model`
**Scope:** document  
**Description:** Every `relationship.to_column` value MUST match the `name` of a column in the model identified by `to_model`.

**Passes:**
```yaml
models:
  - model_id: "customers"
    columns:
      - name: "customer_id"
relationships:
  - to_model: "customers"
    to_column: "customer_id"
```

**Fails:**
```yaml
models:
  - model_id: "customers"
    columns:
      - name: "customer_id"
relationships:
  - to_model: "customers"
    to_column: "id"   # not a column in customers
```

---

#### V-026 — `concepts.maps_to_models` entries resolve to defined models
**Scope:** document  
**Description:** Every entry in `concept.maps_to_models` MUST match a `model_id` present in the `models` list.

**Passes:**
```yaml
models:
  - model_id: "positions"
concepts:
  - concept_id: "net_exposure"
    maps_to_models:
      - "positions"
```

**Fails:**
```yaml
models:
  - model_id: "positions"
concepts:
  - concept_id: "net_exposure"
    maps_to_models:
      - "trades"   # no model with this id
```

---

#### V-027 — `concepts.maps_to_columns` entries resolve to valid model–column pairs
**Scope:** document  
**Description:** Every entry in `concept.maps_to_columns` MUST have a `model` that resolves to a defined `model_id`, and a `column` that resolves to a column name in that model.

**Passes:**
```yaml
models:
  - model_id: "positions"
    columns:
      - name: "net_exposure_usd"
concepts:
  - concept_id: "net_exposure"
    maps_to_columns:
      - model: "positions"
        column: "net_exposure_usd"
```

**Fails:**
```yaml
models:
  - model_id: "positions"
    columns:
      - name: "net_exposure_usd"
concepts:
  - concept_id: "net_exposure"
    maps_to_columns:
      - model: "positions"
        column: "exposure_value"   # no column with this name in positions
```

---

#### V-028 — `metrics.valid_dimensions` entries resolve to valid model–column pairs
**Scope:** document  
**Description:** Every entry in `metric.valid_dimensions` MUST have a `model` that resolves to a defined `model_id` and a `column` that resolves to a column name in that model.

**Passes:**
```yaml
models:
  - model_id: "orders"
    columns:
      - name: "region"
metrics:
  - metric_id: "order_count"
    valid_dimensions:
      - model: "orders"
        column: "region"
```

**Fails:**
```yaml
metrics:
  - metric_id: "order_count"
    valid_dimensions:
      - model: "orders"
        column: "territory"   # no column named territory in orders
```

---

#### V-029 — `metrics.invalid_dimensions` entries resolve to valid model–column pairs
**Scope:** document  
**Description:** Every entry in `metric.invalid_dimensions` MUST have a `model` that resolves to a defined `model_id` and a `column` that resolves to a column name in that model.

**Passes:**
```yaml
models:
  - model_id: "orders"
    columns:
      - name: "order_id"
metrics:
  - metric_id: "order_count"
    invalid_dimensions:
      - model: "orders"
        column: "order_id"
        reason: "Surrogate key."
```

**Fails:**
```yaml
metrics:
  - metric_id: "order_count"
    invalid_dimensions:
      - model: "orders"
        column: "surrogate_pk"   # no column named surrogate_pk in orders
        reason: "Surrogate key."
```

---

#### V-030 — `source_type: user_defined` MUST NOT appear in `semantic.yaml` provenance blocks
**Scope:** document  
**Description:** No `provenance.source_type` value in `semantic.yaml` MUST be `user_defined`. That value is reserved for the effective provenance of corrections applied at serve time. Its presence in `semantic.yaml` indicates the file was edited incorrectly or generated by a non-conforming tool.

**Passes:**
```yaml
# In semantic.yaml:
provenance:
  source_type: llm_inferred
  confidence: 0.72
  human_reviewed: false
```

**Fails:**
```yaml
# In semantic.yaml:
provenance:
  source_type: user_defined   # forbidden in semantic.yaml
  confidence: 1.0
  human_reviewed: true
```

---

#### V-031 — `metrics[].status` MUST NOT appear in `semantic.yaml`
**Scope:** document  
**Description:** The `status` field on metric objects is runtime-computed by the serving layer. Its presence in `semantic.yaml` indicates either manual editing or generation by a non-conforming tool. Any metric object in `semantic.yaml` containing a `status` field MUST be rejected.

**Passes:**
```yaml
# In semantic.yaml:
metrics:
  - metric_id: "total_revenue"
    expression: "SUM(orders.amount)"
    aggregation: sum
    provenance:
      source_type: log_inferred
      confidence: 0.8
      human_reviewed: false
```

**Fails:**
```yaml
# In semantic.yaml:
metrics:
  - metric_id: "total_revenue"
    expression: "SUM(orders.amount)"
    aggregation: sum
    status: active   # MUST NOT appear in semantic.yaml
```

---

#### V-032 — `metrics[].degraded_reason` MUST NOT appear in `semantic.yaml`
**Scope:** document  
**Description:** Like `status`, `degraded_reason` is runtime-computed. Its presence in `semantic.yaml` is a spec violation.

**Passes:**
```yaml
# In semantic.yaml:
metrics:
  - metric_id: "total_revenue"
    aggregation: sum
```

**Fails:**
```yaml
# In semantic.yaml:
metrics:
  - metric_id: "total_revenue"
    aggregation: sum
    degraded_reason: "references suppressed column"   # MUST NOT appear in semantic.yaml
```

---

#### V-033 — `correction_id` unique within `corrections.yaml`
**Scope:** corrections  
**Description:** No two entries in `corrections.yaml` MUST share the same `correction_id`.

**Passes:**
```yaml
corrections:
  - correction_id: "corr_001"
  - correction_id: "corr_002"
```

**Fails:**
```yaml
corrections:
  - correction_id: "corr_001"
  - correction_id: "corr_001"
```

---

#### V-034 — `corrections.source` is a defined value
**Scope:** corrections  
**Description:** Every correction `source` field MUST be one of: `user_defined`, `system`, `llm_suggested`.

**Passes:**
```yaml
corrections:
  - correction_id: "corr_001"
    source: user_defined
```

**Fails:**
```yaml
corrections:
  - correction_id: "corr_001"
    source: admin   # not a defined value
```

---

#### V-035 — `corrections.status` is a defined value
**Scope:** corrections  
**Description:** Every correction `status` field MUST be one of: `approved`, `pending`, `rejected`, `auto_applied`.

**Passes:**
```yaml
corrections:
  - correction_id: "corr_001"
    source: llm_suggested
    status: pending
```

**Fails:**
```yaml
corrections:
  - correction_id: "corr_001"
    source: llm_suggested
    status: awaiting_review   # not a defined value
```

---

#### V-036 — `status: auto_applied` only valid with `source: system`
**Scope:** corrections  
**Description:** A correction with `status: auto_applied` MUST have `source: system`. Any other `source` value combined with `status: auto_applied` is a spec violation.

**Passes:**
```yaml
corrections:
  - correction_id: "corr_001"
    source: system
    status: auto_applied
```

**Fails:**
```yaml
corrections:
  - correction_id: "corr_001"
    source: user_defined
    status: auto_applied   # auto_applied is only valid with source: system
```

---

#### V-037 — `source: user_defined` corrections MUST have `status: approved`
**Scope:** corrections  
**Description:** A human-authored correction (`source: user_defined`) MUST have `status: approved`. A human-authored correction in any other status indicates a data integrity error — humans do not write `pending` or `auto_applied` corrections.

**Passes:**
```yaml
corrections:
  - correction_id: "corr_001"
    source: user_defined
    status: approved
```

**Fails:**
```yaml
corrections:
  - correction_id: "corr_001"
    source: user_defined
    status: pending   # user corrections must be approved
```

---

#### V-038 — `source: llm_suggested` corrections MUST NOT have `status: auto_applied`
**Scope:** corrections  
**Description:** Agent-proposed corrections can only be `pending`, `approved`, or `rejected`. The combination of `source: llm_suggested` and `status: auto_applied` is logically incoherent and represents a spec violation.

**Passes:**
```yaml
corrections:
  - correction_id: "corr_001"
    source: llm_suggested
    status: pending
```

**Fails:**
```yaml
corrections:
  - correction_id: "corr_001"
    source: llm_suggested
    status: auto_applied   # forbidden combination
```

---

#### V-039 — `corrections.target_type` is a defined value
**Scope:** corrections  
**Description:** Every correction `target_type` MUST be one of: `domain`, `model`, `column`, `relationship`, `concept`, `metric`, `query_template`.

**Passes:**
```yaml
corrections:
  - correction_id: "corr_001"
    target_type: column
```

**Fails:**
```yaml
corrections:
  - correction_id: "corr_001"
    target_type: table   # not a defined value
```

---

#### V-040 — `corrections.correction_type` is a defined value
**Scope:** corrections  
**Description:** Every `correction_type` MUST be one of the defined values in Section 8.3.

**Passes:**
```yaml
corrections:
  - correction_id: "corr_001"
    correction_type: description_override
```

**Fails:**
```yaml
corrections:
  - correction_id: "corr_001"
    correction_type: rename   # not a defined value
```

---

#### V-041 — `smif_version` in `corrections.yaml` matches `semantic.yaml`
**Scope:** workspace  
**Description:** The `smif_version` field in `corrections.yaml` MUST be identical to the `smif_version` field in the co-located `semantic.yaml`. A version mismatch indicates the corrections file was created against a different spec version than the semantic model.

**Passes:**
```yaml
# semantic.yaml
smif_version: "0.1.0"

# corrections.yaml
smif_version: "0.1.0"
```

**Fails:**
```yaml
# semantic.yaml
smif_version: "0.1.0"

# corrections.yaml
smif_version: "0.2.0"   # version mismatch
```

---

#### V-042 — Correction `target_id` resolves to a defined object in `semantic.yaml`
**Scope:** workspace  
**Description:** Every correction's `target_id` MUST resolve to an existing object in `semantic.yaml`. For `target_type: column`, the convention is `model_id.column_name` — both parts MUST resolve. For all other types, `target_id` MUST match the corresponding `*_id` field of an existing object.

**Passes:**
```yaml
# semantic.yaml has:
models:
  - model_id: "orders"
    columns:
      - name: "status"

# corrections.yaml:
corrections:
  - target_type: column
    target_id: "orders.status"
```

**Fails:**
```yaml
# semantic.yaml has:
models:
  - model_id: "orders"

# corrections.yaml:
corrections:
  - target_type: column
    target_id: "orders.fulfilment_status"   # column does not exist in orders
```

---

### C.4 SHOULD rules — `W-` series

---

#### W-001 — Model `description` is substantive
**Scope:** document  
**Description:** Every model `description` SHOULD be at least 30 characters and contain at least one sentence. A one-word or empty-string description provides no agent value and likely indicates a generation failure.

**Passes:**
```yaml
models:
  - model_id: "orders"
    description: "One row per customer order. Primary fact table for revenue analysis."
```

**Warns:**
```yaml
models:
  - model_id: "orders"
    description: "Orders."   # too short to be useful
```

---

#### W-002 — Non-identifier columns have substantive descriptions
**Scope:** document  
**Description:** Columns with `role` other than `identifier` SHOULD have a `description` of at least 20 characters. Identifier columns (surrogate keys, UUIDs) often have no useful business description; all other roles represent semantically meaningful data that agents need to interpret.

**Passes:**
```yaml
columns:
  - name: "order_status"
    role: dimension
    description: "Current fulfilment status of the order. Valid values: pending, shipped, delivered, cancelled."
```

**Warns:**
```yaml
columns:
  - name: "order_status"
    role: dimension
    description: "Status."
```

---

#### W-003 — Ambiguous and domain-dependent columns should be human-reviewed
**Scope:** document  
**Description:** Columns with `difficulty: ambiguous` or `difficulty: domain_dependent` SHOULD have `provenance.human_reviewed: true`. These columns carry the highest risk of LLM misinterpretation and benefit most from human validation.

**Passes:**
```yaml
columns:
  - name: "calc_net_exp_usd"
    difficulty: domain_dependent
    provenance:
      human_reviewed: true
```

**Warns:**
```yaml
columns:
  - name: "calc_net_exp_usd"
    difficulty: domain_dependent
    provenance:
      human_reviewed: false   # high-risk column not yet reviewed
```

---

#### W-004 — Low-confidence inferences flagged for review
**Scope:** document  
**Description:** Any field with `provenance.confidence < 0.5` and `provenance.human_reviewed: false` SHOULD trigger a warning. Very low confidence combined with no human review means an agent may be misled by this field.

**Passes:**
```yaml
provenance:
  source_type: llm_inferred
  confidence: 0.38
  human_reviewed: true   # low confidence but reviewed — acceptable
```

**Warns:**
```yaml
provenance:
  source_type: llm_inferred
  confidence: 0.38
  human_reviewed: false   # low confidence and unreviewed
```

---

#### W-005 — Low-cardinality dimension columns have `example_values` populated
**Scope:** document  
**Description:** Columns with `role: dimension` and `cardinality_category: low` SHOULD have a non-empty `example_values` list. These values help agents construct accurate WHERE clauses and avoid incorrect literal values.

**Passes:**
```yaml
columns:
  - name: "order_status"
    role: dimension
    cardinality_category: low
    example_values: ["pending", "shipped", "delivered", "cancelled"]
```

**Warns:**
```yaml
columns:
  - name: "order_status"
    role: dimension
    cardinality_category: low
    example_values: []   # should be populated for low-cardinality dimensions
```

---

#### W-006 — Ratio and percentage metrics declare `additivity: non_additive`
**Scope:** document  
**Description:** Metrics with `aggregation: ratio` SHOULD declare `additivity: non_additive`. A ratio that is silently treated as additive will produce incorrect results when summed across dimensions.

**Passes:**
```yaml
metrics:
  - metric_id: "conversion_rate"
    aggregation: ratio
    additivity: non_additive
```

**Warns:**
```yaml
metrics:
  - metric_id: "conversion_rate"
    aggregation: ratio
    additivity: fully_additive   # ratios are not additively summable
```

---

#### W-007 — All models have a `ddl_fingerprint`
**Scope:** document  
**Description:** Every model SHOULD have a `ddl_fingerprint` value. Without it, the serving layer cannot detect schema drift and confidence decay (Section 7.3) cannot be applied.

**Passes:**
```yaml
models:
  - model_id: "orders"
    ddl_fingerprint: "sha256:7c4e9f..."
```

**Warns:**
```yaml
models:
  - model_id: "orders"
    # ddl_fingerprint absent — drift detection disabled for this model
```

---

#### W-008 — Parameterised `query_templates` declare their parameters
**Scope:** document  
**Description:** A `query_template` whose `sql_template` contains one or more `:param_name` patterns SHOULD have a corresponding `parameters` list entry for each parameter. Undeclared parameters cannot be resolved by agents.

**Passes:**
```yaml
query_templates:
  - template_id: "revenue_by_date"
    sql_template: "SELECT SUM(amount) FROM orders WHERE order_date = :order_date"
    parameters:
      - name: order_date
        data_type: date
        description: "The date to filter on."
```

**Warns:**
```yaml
query_templates:
  - template_id: "revenue_by_date"
    sql_template: "SELECT SUM(amount) FROM orders WHERE order_date = :order_date"
    parameters: []   # :order_date is undeclared
```

---

#### W-009 — Pending corrections exist in workspace
**Scope:** workspace  
**Description:** If `corrections.yaml` contains any entry with `status: pending`, the validator SHOULD emit a warning listing the pending `correction_id` values. This is not an error — pending corrections are valid — but they represent agent-suggested changes that have not been reviewed. Their existence should be surfaced explicitly.

**Passes (no warning):**
```yaml
corrections:
  - correction_id: "corr_001"
    source: user_defined
    status: approved
```

**Warns:**
```yaml
corrections:
  - correction_id: "corr_002"
    source: llm_suggested
    status: pending   # unreviewed agent suggestion — human attention needed
```

---

#### W-010 — `default_time_dimension` references a non-timestamp column
**Scope:** document  
**Description:** A metric's `default_time_dimension` SHOULD reference a column with `role: timestamp`. When it references a column with a different role, this may indicate a modeling error — or a legitimate legacy schema where dates are stored as integers or varchars. The validator cannot distinguish between the two, so this is advisory.

**Suggested remediation:** If the column genuinely represents a time dimension, change its `role` to `timestamp` in `semantic.yaml` (or via a `role_override` correction). If the column is intentionally non-timestamp (e.g. an integer fiscal period), add a `note` to the metric's `description` explaining the non-standard time dimension so agents handle it correctly.

**Passes (no warning):**
```yaml
models:
  - model_id: "orders"
    columns:
      - name: "order_date"
        role: timestamp
metrics:
  - metric_id: "revenue"
    default_time_dimension:
      model: "orders"
      column: "order_date"
```

**Warns:**
```yaml
models:
  - model_id: "orders"
    columns:
      - name: "fiscal_period_int"
        role: dimension   # integer fiscal period — not a spec violation but warrants review
metrics:
  - metric_id: "revenue"
    default_time_dimension:
      model: "orders"
      column: "fiscal_period_int"
```

---

### C.5 Rule index

| ID | Tier | Scope | Short description |
|---|---|---|---|
| V-001 | MUST | document | `smif_version` present and semver |
| V-002 | MUST | document | `generated_at` is valid ISO 8601 UTC |
| V-003 | MUST | document | At least one model present |
| V-004 | MUST | document | Each model has at least one column |
| V-005 | MUST | document | `source.type` is a defined value |
| V-006 | MUST | document | `model_id` unique within document |
| V-007 | MUST | document | `relationship_id` unique within document |
| V-008 | MUST | document | `metric_id` unique within document |
| V-009 | MUST | document | `concept_id` unique within document |
| V-010 | MUST | document | `template_id` unique within document |
| V-011 | MUST | document | Provenance block present on all substantive objects |
| V-012 | MUST | document | `confidence` in range [0.0, 1.0] |
| V-013 | MUST | document | `human_reviewed` is boolean |
| V-014 | MUST | document | `provenance.source_type` is a defined value |
| V-015 | MUST | document | `column.role` is a defined value |
| V-016 | MUST | document | `relationship_type` is a defined value |
| V-017 | MUST | document | `metric.aggregation` is a defined value |
| V-018 | MUST | document | `metric.additivity` is a defined value when present |
| V-019 | MUST | document | `column.cardinality_category` is a defined value |
| V-020 | MUST | document | `column.difficulty` is a defined value when present |
| V-021 | MUST | document | At most one `preferred: true` relationship per model pair |
| V-022 | MUST | document | Relationship `from_model` resolves to a defined model |
| V-023 | MUST | document | Relationship `to_model` resolves to a defined model |
| V-024 | MUST | document | Relationship `from_column` resolves in `from_model` |
| V-025 | MUST | document | Relationship `to_column` resolves in `to_model` |
| V-026 | MUST | document | `concepts.maps_to_models` entries resolve to defined models |
| V-027 | MUST | document | `concepts.maps_to_columns` entries resolve to valid pairs |
| V-028 | MUST | document | `metrics.valid_dimensions` entries resolve to valid pairs |
| V-029 | MUST | document | `metrics.invalid_dimensions` entries resolve to valid pairs |
| V-030 | MUST | document | `source_type: user_defined` forbidden in `semantic.yaml` |
| V-031 | MUST | document | `metrics[].status` forbidden in `semantic.yaml` |
| V-032 | MUST | document | `metrics[].degraded_reason` forbidden in `semantic.yaml` |
| V-033 | MUST | corrections | `correction_id` unique within `corrections.yaml` |
| V-034 | MUST | corrections | `corrections.source` is a defined value |
| V-035 | MUST | corrections | `corrections.status` is a defined value |
| V-036 | MUST | corrections | `status: auto_applied` only valid with `source: system` |
| V-037 | MUST | corrections | `source: user_defined` corrections MUST have `status: approved` |
| V-038 | MUST | corrections | `source: llm_suggested` MUST NOT have `status: auto_applied` |
| V-039 | MUST | corrections | `corrections.target_type` is a defined value |
| V-040 | MUST | corrections | `corrections.correction_type` is a defined value |
| V-041 | MUST | workspace | `smif_version` in `corrections.yaml` matches `semantic.yaml` |
| V-042 | MUST | workspace | Correction `target_id` resolves to a defined object |
| W-001 | SHOULD | document | Model `description` is substantive |
| W-002 | SHOULD | document | Non-identifier columns have substantive descriptions |
| W-003 | SHOULD | document | Ambiguous/domain-dependent columns are human-reviewed |
| W-004 | SHOULD | document | Low-confidence unreviewed fields flagged |
| W-005 | SHOULD | document | Low-cardinality dimensions have `example_values` |
| W-006 | SHOULD | document | Ratio metrics declare `additivity: non_additive` |
| W-007 | SHOULD | document | All models have `ddl_fingerprint` |
| W-008 | SHOULD | document | Parameterised templates declare their parameters |
| W-009 | SHOULD | workspace | Pending corrections surfaced for review |
| W-010 | SHOULD | document | `default_time_dimension` references a non-timestamp column |
---

*SMIF is an open specification. Contributions, issues, and proposals are welcome at https://github.com/strata-spec/spec. The specification is governed by the GOVERNANCE.md document in that repository.*
