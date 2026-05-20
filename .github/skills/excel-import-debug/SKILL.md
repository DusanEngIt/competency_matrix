---
name: excel-import-debug
description: "Debug Excel import jobs that are stuck, failing, or producing wrong results. Use when an import job is in PENDING/FAILED state, rows are being wrongly rejected, skills are not being mapped, or the progress bar is not updating."
argument-hint: "Provide the import job_id or describe what went wrong"
---

# Excel Import Debug

Traces an Excel import job end-to-end through upload → Celery → validation → DB.

## When to Use

- Import job stuck in `PENDING` or `FAILED` status
- Unexpected row rejections (skill mapping failures, validation errors)
- Progress bar frozen (Celery worker issue)
- Skills not appearing after a completed import
- Column mapping not saving correctly

## Procedure

### 1. Check Job Status in DB

```sql
SELECT id, filename, status, total_rows, processed_rows,
       success_rows, error_rows, column_mapping, error_report,
       created_at, completed_at
FROM import_jobs
WHERE id = '<job_id>'
-- or list recent jobs:
ORDER BY created_at DESC LIMIT 10;
```

### 2. Check Celery Worker is Running

```bash
docker compose ps celery
docker compose logs celery --tail=100 | grep -E "ERROR|WARN|job_id"
```

If worker is down:

```bash
docker compose restart celery
```

### 3. Check Redis for Job State

```bash
docker compose exec redis redis-cli keys "*<job_id>*"
```

### 4. Inspect Celery Task Queue

```bash
docker compose exec redis redis-cli llen celery   # pending task count
docker compose exec backend celery -A app.celery inspect active
docker compose exec backend celery -A app.celery inspect reserved
```

### 5. Reproduce Validation Errors

Look at `error_report` JSONB column from step 1. Common causes:

| Error                      | Cause                                  | Fix                                                             |
| -------------------------- | -------------------------------------- | --------------------------------------------------------------- |
| `skill_not_found`          | Skill name too different from taxonomy | Re-check AI embedding service; lower similarity threshold       |
| `employee_not_found`       | Email mismatch                         | Verify email column mapping; check `employees` table            |
| `proficiency_out_of_range` | Text level not in map                  | Add mapping: `Junior→2, Mid→3, Senior→4` in `services/excel.py` |
| `duplicate_row`            | Same employee+skill twice              | Expected — row deduplicated automatically                       |

### 6. Manually Retry a Failed Job

```bash
docker compose exec backend python -c "
from app.tasks.import_task import process_import
process_import.delay('<job_id>')
"
```

### 7. Check AI Skill Mapping (if skills unmapped)

```bash
curl -s -X POST http://localhost:5000/embed \
  -H 'Content-Type: application/json' \
  -d '{"text": "<unknown_skill_name>"}' | jq '.embedding | length'
# Should return 384
```

Then verify similarity against taxonomy in pgvector:

```sql
SELECT name, 1 - (embedding <=> (
  SELECT embedding FROM skills WHERE name = '<known_skill>'
)) AS similarity
FROM skills
ORDER BY similarity DESC
LIMIT 5;
```

## Output

Report:

1. Job status + row counts
2. Whether Celery is healthy
3. Top error categories from `error_report`
4. Identified root cause
5. Exact fix steps (config change, retry command, or data correction needed)
