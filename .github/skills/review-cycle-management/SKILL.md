---
name: review-cycle-management
description: "Manage employee skill review cycles: create semi-annual or annual cycles, check completion status, send or resend reminders, handle manager sign-off, extend deadlines, and finalize or close cycles. Use when HR needs to start a cycle, check who hasn't completed a review, or force-close a stale cycle."
argument-hint: "Describe the task: 'start new cycle', 'check completion status', 'extend deadline', 'force close cycle', or provide cycle_id"
---

# Review Cycle Management

Operational guide for managing review cycles end-to-end.

## Cycle Types

| Type          | Trigger                          | Sign-off Required     |
| ------------- | -------------------------------- | --------------------- |
| `SEMI_ANNUAL` | HR creates manually or scheduled | No                    |
| `ANNUAL`      | HR creates manually              | Manager must validate |
| `PROJECT`     | HR or Manager triggers manually  | No                    |

## Create a New Cycle

```bash
curl -X POST http://localhost:8000/api/reviews \
  -H "Authorization: Bearer <HR_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "H1 2026 Skill Review",
    "type": "SEMI_ANNUAL",
    "start_date": "2026-06-01",
    "end_date": "2026-06-30"
  }'
```

This creates `employee_review_status` rows (status=`PENDING`) for all active employees.

## Check Completion Status

```sql
-- Overall completion rate
SELECT
  COUNT(*) AS total,
  COUNT(*) FILTER (WHERE status = 'COMPLETED') AS completed,
  COUNT(*) FILTER (WHERE status = 'PENDING') AS pending,
  ROUND(COUNT(*) FILTER (WHERE status = 'COMPLETED') * 100.0 / COUNT(*), 1) AS pct_done
FROM employee_review_status
WHERE review_cycle_id = '<cycle_id>';

-- Who hasn't completed
SELECT e.email, e.department, e.title, ers.status, ers.completed_at
FROM employee_review_status ers
JOIN employees e ON e.id = ers.employee_id
WHERE ers.review_cycle_id = '<cycle_id>'
  AND ers.status = 'PENDING'
ORDER BY e.department, e.last_name;
```

## Send / Resend Reminders

Reminders are auto-sent at T-14 days via Celery beat. To send manually:

```bash
docker compose exec backend python -c "
from app.tasks.review_reminders import send_reminders
send_reminders.delay('<cycle_id>')
"
```

## Extend a Deadline

```sql
UPDATE review_cycles
SET end_date = '<new_end_date>'
WHERE id = '<cycle_id>' AND status = 'ACTIVE';
```

Then invalidate any cached reminder state:

```bash
docker compose exec redis redis-cli keys "review:*" | xargs docker compose exec redis redis-cli del
```

## Manager Sign-off (Annual Cycles)

```sql
-- Check which manager validations are outstanding
SELECT e.email AS employee, m.email AS manager, ers.manager_validated_at
FROM employee_review_status ers
JOIN employees e ON e.id = ers.employee_id
JOIN employees m ON m.id = e.manager_id
WHERE ers.review_cycle_id = '<cycle_id>'
  AND ers.status = 'COMPLETED'
  AND ers.manager_validated_at IS NULL;
```

## Force Close a Stale Cycle

```sql
-- Mark all remaining PENDING as skipped, then close the cycle
UPDATE employee_review_status
SET status = 'SKIPPED'
WHERE review_cycle_id = '<cycle_id>' AND status = 'PENDING';

UPDATE review_cycles
SET status = 'CLOSED'
WHERE id = '<cycle_id>';
```

## Output

After any cycle operation, provide:

1. Cycle name, type, dates, current status
2. Completion rate (total / completed / pending)
3. List of departments with lowest completion
4. Next recommended action
