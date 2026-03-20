# Governance

## Current status

SMIF is in pre-ratification draft (`0.1.0-draft`). During this phase the specification is maintained by a single author. Decisions are made by the author. This is intentional — a spec in early iteration needs to move fast and maintain internal consistency, which committees are bad at.

## How decisions are made now (v0.x)

All spec changes require a GitHub issue before a pull request. The issue must state:

1. The specific field, rule, or section being changed
2. The field inclusion test result: does this change affect what SQL an agent generates?
3. At least one real-world case that motivates the change

The author will respond to every substantive issue. Response means one of: accepted, rejected with rationale, or deferred to a named future version. Silence is not a response; if an issue receives no response within 30 days, open a follow-up.

Pull requests without a preceding issue will be closed without review, except for:
- Typo and grammar fixes
- Example file additions (new domains in `examples/`)
- JSON Schema skeleton completions that do not change the spec text

## Contributions

The easiest contribution is a new example. If you have a real Postgres schema and have generated a valid `semantic.yaml` against it, open a PR adding it under `examples/your_domain/`. Include all three files (`strata.md`, `semantic.yaml`, `corrections.yaml`). Examples do not need to be perfect — annotate known weaknesses in the `strata.md`.

See `CONTRIBUTING.md` for full contribution guidance.

## Transition to foundation governance

At v1.0, if the following conditions are met, governance transfers to a neutral foundation:

1. At least two independent implementations exist (tools not built by the original author)
2. At least two organisations outside the core team have adopted SMIF in production
3. The community has expressed interest in formal governance through GitHub issues or discussion

The foundation will be chosen at that time based on community preference. Candidates include the Linux Foundation, OpenJS Foundation, and CNCF. The author commits not to block this transition.

Until those conditions are met, unilateral maintainer governance is the honest description of how this works — and this document says so explicitly rather than pretending otherwise.

## Spec versioning policy

- **Patch versions** (0.1.x): Clarifications, example additions, non-normative text changes. No changes to required fields, enum values, or validation rules.
- **Minor versions** (0.x.0): New optional fields, new enum values, new validation rules. Existing valid documents remain valid.
- **Major versions** (x.0.0): Breaking changes. Removals, type changes, renamed fields. Every major version bump ships with a migration guide and, where possible, a migration tool.

The changelog in `CHANGELOG.md` records every change. The validation rule identifiers in Appendix C of the spec are stable within a minor version — a rule ID means the same thing across all patch releases of a given minor version.
