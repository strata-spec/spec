# E-commerce Example — Annotation Decisions

This document explains non-obvious annotation choices in `semantic.yaml` and `corrections.yaml`.
It is intended for contributors who want to understand the reasoning behind specific field
values, and as a guide for authoring similar SMIF documents for other domains.

---

## 1. Why `orders.total_amount` is corrected to `role: dimension`

The column is a `numeric` type and would normally be classified as `role: measure`. The
correction demotes it to `dimension` because the field is unreliable for aggregation: it is
populated at checkout and does not reflect post-checkout adjustments (promotions, partial
refunds applied as credits). The canonical revenue figures come from `order_items`, where
line-level prices are immutable.

This demonstrates the correction type `role` — correcting a structural inference without
suppressing the column entirely. The column remains available for display (e.g., showing a
customer their receipt total), but agents will not attempt to SUM it.

---

## 2. Why `customers.customer_since` uses `difficulty: context_dependent`

The column name is unambiguous — the values are dates. The difficulty is not in understanding
what is stored, but in knowing when to use it versus `orders.created_at`. Agents reliably
confuse "when did this customer register?" with "when did this customer first buy?". The
`context_dependent` difficulty level signals that additional context (i.e., the description)
is required before an agent should use this column in a time dimension.

---

## 3. Why `orders.total_amount` uses `difficulty: ambiguous`

`ambiguous` is the strongest difficulty signal. It is warranted here because the column name
implies it is the revenue figure, but it systematically disagrees with the correct revenue
calculation in a way that is invisible at query time — there is no error, just a wrong number.
Agents need an explicit warning that this column is not the source of truth.

---

## 4. Why `revenue` and `active_customer_count` metrics use `source_type: user_defined`

These metric definitions did not come from query logs or LLM inference — they were explicitly
authored by the data team based on internal business definitions. `user_defined` with
`confidence: 1.0` and `human_reviewed: true` gives consumer agents the strongest possible
signal that these definitions should be trusted without question.

In contrast, `order_count` is `log_inferred` (0.88 confidence) because the definition was
derived from the most common pattern in the query log, and a human has not yet formally
reviewed whether it matches the business definition exactly.

---

## 5. Why `gross_margin` has two `required_filters`

The first filter (`orders.status NOT IN ('cancelled', 'refunded')`) is needed for all
revenue-derived metrics. The second filter (`order_items.unit_cost IS NOT NULL`) is specific
to margin calculations. Including both in the metric definition means a consuming agent never
needs to infer them — they are stated explicitly as non-negotiable preconditions.

This is the intended use of `required_filters` at the metric level (as introduced in SMIF
0.1.1): encoding business rules that must always be applied, rather than leaving them to
the agent to discover from the description text.

---

## 6. Provenance source_type coverage

This example uses all seven `source_type` values defined in the spec:

| `source_type`       | Where used |
|---------------------|------------|
| `user_defined`      | `customers.country_code`, `customers.tier` (corrections), `revenue` metric, `active_customer_count` metric, concept definitions |
| `ddl_comment`       | `customers.email`, `customers.name`, `products.sku` |
| `schema_constraint` | Primary keys, foreign keys, `orders.created_at`, model-level provenance |
| `catalog_import`    | `orders.shipping_country_code` |
| `code_extracted`    | `customers.tier` (extracted from CRM job source code) |
| `log_inferred`      | `products.category`, `orders.status`, `order_count` metric |
| `llm_inferred`      | `customers.customer_since`, `products.name`, `products.price`, `orders.total_amount`, `order_items.unit_price`, `order_items.unit_cost`, `inventory.quantity_on_hand`, `gross_margin` metric, domain block |

The variety of provenance types in a single document reflects real-world conditions: some
fields are perfectly specified by schema constraints, others require LLM inference, and a
few are known definitively from source code or the internal data catalogue.

---

## 7. Why `orders_to_customers` has `always_valid: false`

The relationship is not always valid because `customer_id` is NULL for guest checkouts (~22%
of orders). A join on this relationship will drop those rows silently. The `always_valid: false`
flag tells consumer agents to be aware of the data loss and to offer the user a choice about
whether to include or exclude guest orders.

Contrast this with `order_items_to_orders`, which has `always_valid: true` — every order item
references a valid order header, with no NULLs or orphaned rows.

---

## 8. Why `customers.tier` uses `source_type: code_extracted`

The `valid_values` list for the `tier` column (`standard`, `silver`, `gold`, `vip`) was
extracted directly from the CRM tier assignment job's source code, where these values are
defined as an enum. `code_extracted` with `confidence: 0.95` (not 1.0) acknowledges a small
residual risk that the source code has diverged from the database without the annotation being
updated. A human review would confirm this.

---

## 9. Why the `concepts` layer is used for `revenue` and `gross_margin`

The concepts layer provides a named, cross-model abstraction that can be referenced without
knowing which specific columns or models implement it. This is useful when the same business
concept is queried in multiple contexts — for example, "revenue" might be computed differently
for different report types. Defining it as a concept anchors the definition and prevents
agent drift toward alternative (incorrect) interpretations.

The `broader` field on `gross_margin_concept` references `revenue_concept`, expressing the
relationship "gross margin is a refinement of the revenue concept" — a useful signal for
agents reasoning about which metric to use when a user's question is ambiguous.

---

## 10. Corrections file structure

All corrections in this example use `source: user_defined` as required by the spec's
corrections provenance contract (Section 8.5 of SPEC.md). The two required correction types
are demonstrated:

- **`correction_type: description`** — `corr_001` (unit_cost) and `corr_003` (customer_since)
  replace LLM-generated descriptions with analyst-verified ones.
- **`correction_type: role`** — `corr_002` (total_amount) changes the structural role from
  `measure` to `dimension`, preventing revenue aggregation on an unreliable column.
- **`correction_type: suppress`** — `corr_004` and `corr_005` remove PII columns from the
  served semantic model entirely.
