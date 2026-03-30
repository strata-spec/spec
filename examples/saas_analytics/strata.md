# Database: SaaS Product Analytics

## What this database is for

Product analytics database for a B2B SaaS application. Tracks user behaviour,
subscription lifecycle, and feature adoption. Primary consumers are the growth
team (acquisition and activation funnels), finance (MRR and churn reporting),
and product (feature engagement dashboards).

The database is append-heavy. The `events` table is a wide table — all
interaction types share one physical table, discriminated by `event_type`.
Subscriptions are preserved historically; ended subscriptions remain in the
table with `ended_at` set. Users are never hard-deleted; churned users have
`churned_at` set.

## Key business concepts

- **Event**: A single interaction recorded in the product. All events share the
  `events` table, discriminated by `event_type`. Always filter on `event_type`
  before querying events — mixing event types produces semantically incoherent
  result sets.
- **Session**: A contiguous window of user activity. Sessions are computed
  server-side with a 30-minute inactivity timeout. A single user may have
  multiple sessions per day.
- **Daily Active User (DAU)**: A user who generates at least one `user_active`
  sentinel event in a calendar day. DAU is computed from the `events` table
  filtered to `event_type = 'user_active'`.
- **Monthly Recurring Revenue (MRR)**: The normalised monthly value of all
  active subscriptions at a point in time. MRR is a semi-additive metric —
  you can sum across users but not across time; summing across months gives
  cumulative contract value, not average MRR.
- **Churn**: A user who was on a paid plan and cancelled. Recorded by
  `users.churned_at` (set at cancellation) and by the subscription row
  transitioning to `status = 'cancelled'`.
- **Plan**: A pricing tier that governs features and pricing. Plans are
  reference data; a user holds exactly one `plan_id` at a time (current plan
  only — plan history is tracked via the subscriptions table).

## Known gotchas

- `events` is a wide table. `feature_name` and `page_name` are only meaningful
  for `event_type = 'feature_used'` and `event_type = 'page_view'` respectively.
  They are NULL for all other event types. Always filter `event_type` first.
- DAU must be computed from `event_type = 'user_active'`. Do not count distinct
  `user_id` across all event types — that inflates DAU with bot traffic and
  system events.
- MRR is stored per subscription row (`subscriptions.mrr_usd`). For current
  MRR, filter `subscriptions.status = 'active'`. Summing `mrr_usd` across all
  rows (including cancelled) gives total-contract-value, not MRR.
- `users.plan_id` is the user's current plan. For plan history, query the
  `subscriptions` table. Do not use `users.plan_id` in historical MRR reports.
- `sessions.ended_at` is NULL for sessions currently in progress. Exclude
  in-progress sessions when computing duration statistics unless explicitly
  asked for live session counts.
- `subscriptions.ended_at` is NULL for active subscriptions. Do not filter
  `ended_at IS NOT NULL` when you want active subscriptions — filter on
  `status = 'active'` instead.
- Timestamps are stored in UTC. All `occurred_at`, `started_at`, `ended_at`,
  and `created_at` columns are `TIMESTAMPTZ` in UTC. Convert to the user's
  local timezone only when displaying to end-users.

## Sensitive columns

- `saas.users.email` — personal data, do not include in aggregated outputs
- `saas.users.user_id` — do not expose in public-facing dashboards

## Do not infer metrics from

- `saas.audit_log` — internal audit trail, not a business table
- `saas.feature_flags` — runtime flag configuration, not analytics data
