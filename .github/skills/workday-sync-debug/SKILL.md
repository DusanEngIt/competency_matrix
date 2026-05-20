---
name: workday-sync-debug
description: "Debug Workday sync failures, employees not syncing, org chart not updating, or new hires not being created. Use when the daily sync task fails, employee data is stale, manager relationships are wrong, or terminated employees are still active."
argument-hint: "Describe the sync issue: e.g. 'sync failed last night', 'new hire not appearing', 'manager_id not updating'"
---

# Workday Sync Debug

Traces the daily Workday → platform employee sync end-to-end.

## When to Use

- Daily sync task failed or timed out
- New hires not appearing in the platform after next sync
- Terminated employees still showing as active (`is_active = TRUE`)
- Manager relationships (`manager_id`) incorrect or not updated
- Sync ran but employee data (department, title) not updated

## Sync Overview

```text
Direction: Workday → Platform (one-way)
Schedule:  Daily at 02:00 (Celery beat)
Key field: workday_id (upsert key — never employee email)
On termination: is_active = FALSE (never hard-delete)
```

## Procedure

### 1. Check Last Sync Run

```sql
SELECT id, status, created_at, completed_at,
       total_rows, success_rows, error_rows, error_report
FROM workday_sync_jobs
ORDER BY created_at DESC
LIMIT 5;
```

If no recent row: Celery beat is not running or the task was never scheduled.

### 2. Check Celery Beat is Running

```bash
docker compose ps | grep celery
docker compose logs celery --tail=50 | grep "workday\|beat\|crontab"
```

If beat is not running:

```bash
# Beat must run as a separate process or be included in Celery startup
docker compose exec backend celery -A app.celery beat --loglevel=info &
```

### 3. Check Workday API Credentials

```bash
docker compose exec backend env | grep WORKDAY
# WORKDAY_API_URL, WORKDAY_CLIENT_ID, WORKDAY_CLIENT_SECRET must be set
```

Test connectivity:

```bash
docker compose exec backend python -c "
from app.services.workday import get_workday_token
token = get_workday_token()
print('Token acquired:', bool(token))
"
```

### 4. Manually Trigger a Sync

```bash
docker compose exec backend python -c "
from app.tasks.workday_sync import sync_all_employees
sync_all_employees.delay()
print('Sync task queued')
"
```

Then tail logs:

```bash
docker compose logs celery -f | grep "workday\|sync\|ERROR"
```

### 5. Verify Sync Results

```bash
# Employee count before/after
SELECT COUNT(*) FROM employees WHERE is_active = TRUE;

# Last sync timestamp per employee
SELECT workday_id, email, updated_at
FROM employees
ORDER BY updated_at DESC
LIMIT 20;

# Recently terminated employees
SELECT email, updated_at
FROM employees
WHERE is_active = FALSE
ORDER BY updated_at DESC
LIMIT 10;
```

### 6. Diagnose API Error Types

| Error                         | Likely Cause                           | Fix                                                      |
| ----------------------------- | -------------------------------------- | -------------------------------------------------------- |
| `401 Unauthorized`            | Expired or wrong credentials           | Rotate `WORKDAY_CLIENT_SECRET` in `.env` + restart       |
| `429 Too Many Requests`       | Rate limit hit                         | Add delay between API pages; check Workday rate limits   |
| `Connection timeout`          | Workday API unreachable                | Check network/firewall from VM; verify `WORKDAY_API_URL` |
| `KeyError: 'workday_id'`      | API response schema changed            | Update `services/workday.py` field mapping               |
| Sync completes but no updates | `workday_id` mismatch (UUID vs string) | Normalize workday_id format in sync service              |

### 7. Check Org Chart Updates

```sql
-- Employees without a manager (should only be top-level)
SELECT email, title FROM employees
WHERE manager_id IS NULL AND is_active = TRUE;

-- Manager relationship sanity check
SELECT e.email, m.email AS manager_email
FROM employees e
LEFT JOIN employees m ON m.id = e.manager_id
WHERE e.is_active = TRUE
LIMIT 20;
```

## Output

Report:

1. Last sync job status and timestamps
2. Celery beat running (yes/no)
3. API connectivity result
4. Error category from logs/error_report
5. Affected employee count
6. Root cause + fix steps
