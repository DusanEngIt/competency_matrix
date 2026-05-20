---
name: celery-task-debug
description: "Debug any Celery task that is stuck, failing, or not running. Use when a task is in PENDING state, a worker is unresponsive, a review reminder was not sent, an embedding rebuild did not complete, or a Workday sync task timed out. General Celery diagnostics — for import/export-specific issues use excel-import-debug or excel-export-debug."
argument-hint: "Describe what task or job appears stuck, or paste the job_id / task name"
---

# Celery Task Debug

Diagnoses Celery task failures, worker outages, queue backlogs, and timeout issues.

## When to Use

- Task stuck in `PENDING` indefinitely
- Worker unresponsive or showing no activity
- Task ran but did not produce expected results (no DB update, no email sent)
- Review reminders not sent
- Profile embedding rebuild not completing after skill edit
- Workday sync task timed out or silently failed

## Procedure

### 1. Check Worker is Running

```bash
docker compose ps celery
docker compose logs celery --tail=50
```

If not running:

```bash
docker compose up -d celery
```

### 2. Check Queue Depth

```bash
# Number of tasks waiting in default queue
docker compose exec redis redis-cli llen celery

# Number in each named queue
docker compose exec redis redis-cli keys "celery*" | sort
```

### 3. Inspect Active and Reserved Tasks

```bash
docker compose exec backend celery -A app.celery inspect active
docker compose exec backend celery -A app.celery inspect reserved
docker compose exec backend celery -A app.celery inspect scheduled
```

### 4. Check for Failed Tasks

```bash
docker compose exec backend celery -A app.celery events --dump
docker compose logs celery --tail=200 | grep -E "ERROR|FAILURE|Traceback|retry"
```

### 5. Identify Task by Name

| Task                                           | File                        | Triggered by               |
| ---------------------------------------------- | --------------------------- | -------------------------- |
| `import_task.process_import`                   | `tasks/import_task.py`      | Excel import confirm       |
| `export_task.process_export`                   | `tasks/export_task.py`      | Export request             |
| `embed_profile.rebuild_profile_embedding`      | `tasks/embed_profile.py`    | Profile skill edit         |
| `workday_sync.sync_all_employees`              | `tasks/workday_sync.py`     | Daily cron 02:00           |
| `review_reminders.send_reminders`              | `tasks/review_reminders.py` | T-14 days before cycle end |
| `notifications.send_skill_change_notification` | `tasks/notifications.py`    | Manager edits subordinate  |

### 6. Manually Re-run a Task

```bash
docker compose exec backend python -c "
from app.tasks.<module> import <task_name>
<task_name>.delay('<arg>')
"
```

### 7. Purge Stuck Queue (last resort)

```bash
# WARNING: discards all pending tasks
docker compose exec backend celery -A app.celery purge
```

### 8. Check for Poison Tasks (task that always crashes worker)

```bash
# If worker restarts in a loop, identify crashing task:
docker compose logs celery | grep "SIGTERM\|worker lost\|Killed"
# Then purge and investigate task args
```

## Common Root Causes

| Symptom                        | Cause                          | Fix                                        |
| ------------------------------ | ------------------------------ | ------------------------------------------ |
| Queue depth grows, no progress | Worker down                    | `docker compose restart celery`            |
| Task runs, no DB update        | Sync session error inside task | Check for async session use (must be sync) |
| Task immediately fails         | Missing env var / secret       | Check `docker compose exec celery env`     |
| Worker restart loop            | OOM or poison task             | Increase container memory; purge queue     |
| Scheduled task never runs      | Celery beat not running        | Add `celery -A app.celery beat` to compose |

## Output

Report:

1. Worker status (running / down)
2. Queue depth
3. Task identified and its trigger
4. Error from logs
5. Root cause + specific fix command
