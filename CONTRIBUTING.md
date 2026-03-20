# Contributing to SMIF

## The contribution gate

Every proposed addition to the core spec must pass this test before a pull request is opened:

> **Does this field change what SQL an agent generates, or change whether an agent should generate SQL at all?**

If the answer is no, the field does not belong in the spec. It may belong in the `properties` escape hatch on the relevant object type. This test is not a bureaucratic hurdle — it is the design principle that keeps the format small enough to be useful.

---

## Types of contribution

### Adding an example (easiest — PR directly, no issue needed)

If you have a real Postgres schema and have used it to generate a valid SMIF document, we want it. Open a PR adding a new directory under `examples/`:

```
examples/your_domain/
  strata.md          # The human-intent prior you wrote
  semantic.yaml      # The generated semantic model
  corrections.yaml   # Any corrections you applied
```

**Requirements for example PRs:**
- All three files must be present
- `semantic.yaml` must pass `strata validate` (all V- rules) when the validator is available. Until then, manually check against the field definitions in `SPEC.md`
- If any fields have been sanitised or anonymised, note this in the `strata.md`
- Examples do not need to be perfect. Annotate known weaknesses, ambiguous columns, and places where the inference got it wrong. Real examples with honest annotations are more valuable than sanitised showcases

### Reporting a spec gap (issue only)

A spec gap is a situation where you encountered a real schema and found that SMIF could not represent something you needed to represent — not because the format is wrong, but because it is missing something.

Open an issue with:
1. A description of the schema pattern you encountered
2. What you needed to represent and could not
3. A proposed field or structure, with the field inclusion test result
4. An example of what the YAML would look like if your proposal were accepted

The author will respond within 30 days.

### Proposing a field addition (issue required before PR)

1. Open an issue applying the field inclusion test
2. Wait for the issue to be acknowledged and marked `accepted-for-pr`
3. Open a PR that modifies `SPEC.md` and adds or updates at least one example in `examples/`
4. The PR must also update:
   - The relevant field table in the schema reference section
   - Appendix B (changelog) with the new field
   - Appendix C if the new field requires a new validation rule

PRs that change `SPEC.md` without a preceding accepted issue will be closed without review.

### Proposing a new validation rule (issue required before PR)

New validation rules follow the same issue-first process. Additionally:

- Every proposed rule must include a passing example and a failing example (matching the format in Appendix C)
- Proposed MUST rules must satisfy the test: is this always a spec error, or just sometimes a bad practice? If sometimes, it belongs in SHOULD
- SHOULD rules must include a remediation hint — a warning without a suggested fix is noise

### Fixing typos and clarifying non-normative text (PR directly)

Grammar fixes, typo corrections, and clarifications to non-normative prose do not require an issue. Open a PR directly. Mark it `[non-normative]` in the title.

---

## What belongs in the spec vs in a tool

The spec defines the format. The spec does not define:
- How a tool generates the format (inference algorithms, LLM prompts, query log mining strategies)
- How a tool serves the format (MCP server implementation, API design)
- How a tool validates the format beyond the rules in Appendix C
- Access control, multi-tenancy, or user authentication

If your contribution is really about how a tool should behave rather than what a document should contain, it belongs in the tool's documentation, not in this spec.

---

## Code of conduct

Be specific, be evidence-based, be direct. Disagreements about spec decisions are welcome and productive. Personal criticism of contributors is not.

The author has final say on spec changes during v0.x. If you disagree with a decision, open a counter-issue with your reasoning. The author will respond. If you believe the process itself is unfair, raise it in a GitHub Discussion.
