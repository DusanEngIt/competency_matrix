---
name: excel-export-debug
description: "Debug Excel export jobs that are stuck, failing, or producing empty/incorrect files. Use when an export job is in PENDING/FAILED state, the download link never appears, the generated file is missing rows, or export scope is wrong (HR seeing only own data, manager seeing all employees, etc.)."
argument-hint: "Provide the export job_id or describe what went wrong"
---

# Excel Export Debug

Traces an Excel export job end-to-end through request → Celery → scope filter → file generation → download.

## When to Use

- Export job stuck in `PENDING` or `FAILED` status
- Download link never appears after job completes
- Generated `.xlsx` is empty or missing employees/skills
- Wrong scope returned (e.g., HR only sees own profile, manager sees full workforce)
- File is corrupted or cannot be opened

## Procedure

### 1. Check Job Status in DB

```sql
SELECT id, status, created_by, error_report, created_at, completed_at
FROM export_jobs
WHERE id = '<job_id>'
-- or list recent jobs:
ORDER BY created_at DESC LIMIT 10;
```

### 2. Check Celery Worker is Running

```bash
docker compose ps celery
docker compose logs celery --tail=100 | grep -E "ERROR|WARN|export"
```

If worker is down:

```bash
docker compose restart celery
```

### 3. Verify Scope Filter by Role

The export scope is enforced server-side from the JWT — confirm the requesting user's role:

```sql
SELECT id, email, role FROM employees WHERE id = '<created_by_id>';
```

Expected scope per role:

| Role                 | Expected Filter                        |
| -------------------- | -------------------------------------- |
| `EMPLOYEE`           | `WHERE e.id = <created_by_id>`         |
| `LINE_MANAGER`       | `WHERE e.manager_id = <created_by_id>` |
| `HR_COORDINATOR`     | No filter — full workforce             |
| `GENERAL_MANAGEMENT` | Should be blocked (403)                |

If rows are missing, confirm the filter applied matches the above.

### 4. Check Row Count vs Expected

```sql
-- Expected rows for HR export:
SELECT COUNT(*) FROM employees WHERE is_active = TRUE;

-- Expected rows for a manager's export:
SELECT COUNT(*) FROM employees WHERE manager_id = '<manager_id>' AND is_active = TRUE;
```

Compare against `total_rows` in the `export_jobs` record.

### 5. Check for openpyxl Errors

```bash
docker compose logs celery --tail=200 | grep -E "openpyxl|xlsx|export_task|KeyError|AttributeError"
```

Common causes:

| Error                           | Cause                                    | Fix                                                   |
| ------------------------------- | ---------------------------------------- | ----------------------------------------------------- |
| `KeyError: 'proficiency_level'` | Skill row missing field                  | Check `employee_skills` JOIN in export query          |
| Empty file                      | Scope filter returns 0 rows              | Verify `manager_id` relationship in `employees` table |
| `FAILED` with no `error_report` | Celery task crashed before writing error | Check celery logs directly                            |
| File generated but download 404 | File path not stored or wrong base URL   | Check `file_path` column in `export_jobs`             |

### 6. Manually Retry a Failed Job

```bash
docker compose exec backend python -c "
from app.tasks.export_task import process_export
process_export.delay('<job_id>')
"
```

### 7. Verify Download Endpoint

```bash
curl -I -H "Authorization: Bearer <token>" \
  http://localhost:8000/api/export/<job_id>/download
# Expect: 200 with Content-Disposition: attachment; filename=export_*.xlsx
# If 404: job not COMPLETED yet, or file_path not set
# If 403: role not allowed to download this job
```

## Output

Report:

1. Job status + `created_by` role
2. Scope filter applied vs expected
3. Row count comparison (expected vs exported)
4. Top errors from Celery logs or `error_report`
5. Root cause + fix steps
