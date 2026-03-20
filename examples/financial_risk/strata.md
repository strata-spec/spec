# Database: Risk Management Platform

## What this database is for

Internal risk management database for an equities trading division. Tracks trader positions,
instrument valuations, desk-level exposure limits, and daily P&L. Primary consumers are the
risk team (daily limit monitoring), compliance (breach reporting), and the internal analytics
platform (exposure dashboards).

The database is append-heavy. Historical positions are preserved. The `deleted_at` soft-delete
pattern is used on `positions`, `instruments`, and `desk_limits`. Always filter
`deleted_at IS NULL` when you want current data.

## Key business concepts

- **Position**: A trader's current holding in a specific instrument, expressed as a quantity
  and a calculated net exposure in USD. One row per trader-instrument-business_date.
- **Net Exposure**: The sum of long and short positions after netting within the same instrument
  on the same desk. Long = positive USD, short = negative USD. Always expressed in USD regardless
  of instrument native currency.
- **Desk**: An organisational unit that groups traders and carries collective exposure limits.
- **Limit**: A desk-level maximum net exposure for a given effective date. Limits may change
  daily. Always join on both `desk_id` and the relevant date field.
- **Breach**: A condition where a desk's net exposure in an instrument exceeds its applicable
  limit for that business date. The breach amount is `net_exposure - limit_usd`.
- **Business Date**: The London close date that a position or P&L figure is attributed to.
  Stored as `DATE` in all tables. All timestamps are stored in UTC; `business_date` reflects
  London market close regardless of when the record was written.

## Known gotchas

- `instrument_id` in `positions` is an internal surrogate key. It does NOT correspond to ISIN,
  Bloomberg ticker, SEDOL, or any external identifier. To get external identifiers, join to
  `instruments`.
- `positions.deleted_at IS NULL` must always be appended. Deleted positions are soft-deleted,
  never removed. The `deleted_at` column is NOT indexed; always filter it last.
- P&L figures (`pnl_usd`) are always in USD regardless of instrument `currency`. No conversion
  is needed. Do not multiply by FX rates.
- `desk_limits` are effective per `effective_date`, not a range. To find the limit applicable
  on a given business date, join on `desk_id = desk_id AND effective_date = :business_date`.
  There is no `valid_to` column. If no row exists for a desk on a given date, that desk had
  no formal limit recorded.
- `traders.is_active` is updated in real-time. A trader who was active on a historical
  `business_date` may now show `is_active = false`. Do not filter on `is_active` when
  running historical queries.
- The `desks` table has no `deleted_at`. Decommissioned desks are flagged by
  `desks.is_active = false`. Always join to `desks` for the desk name; do not use
  `positions.desk_id` as a display value.

## Sensitive columns

- `risk.traders.trader_name` — personal data, do not expose in aggregated outputs
- `risk.traders.email` — personal data
- `risk.positions.trader_id` — do not include in any output visible to other traders' desks

## Do not infer metrics from

- `risk.audit_log` — internal audit trail, not a business table
- `risk.migration_history` — schema migration metadata
