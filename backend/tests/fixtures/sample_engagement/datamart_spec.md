# Datamart Spec

## retention_weekly

- grain: one row per user per week
- primary keys: `user_id`, `week_start`
- core columns:
  - `is_retained`
  - `segment_name`
  - `signup_week`
  - `last_active_at`

## Intended Use

- retention trend analysis
- segment comparison
- reactivation analysis
