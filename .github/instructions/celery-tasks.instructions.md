---
description: "Use when writing Celery tasks, background jobs, or async workers in apps/backend/. Covers task structure, retry logic, error handling, state reporting, logging, and progress tracking for long-running jobs."
applyTo: "apps/backend/app/tasks/**"
---

# Celery Task Conventions

## Task File Template

```python
from celery import shared_task
from celery.utils.log import get_task_logger
from app.database import get_sync_session   # use sync session inside Celery

logger = get_task_logger(__name__)

@shared_task(
    bind=True,
    max_retries=3,
    default_retry_delay=60,        # seconds before first retry
    acks_late=True,                # re-queue on worker crash
    reject_on_worker_lost=True,
)
def my_task(self, job_id: str) -> dict:
    logger.info("Starting task", extra={"job_id": job_id})
    try:
        # ... work ...
        return {"status": "ok"}
    except SomeTransientError as exc:
        logger.warning("Transient error, retrying", extra={"exc": str(exc)})
        raise self.retry(exc=exc, countdown=60 * (2 ** self.request.retries))
    except Exception as exc:
        logger.error("Permanent failure", extra={"job_id": job_id, "exc": str(exc)})
        _mark_job_failed(job_id, str(exc))
        raise   # do NOT swallow — let Celery mark task as FAILURE
```

## Progress Reporting Pattern

For jobs with a status row in the DB (import_jobs, export_jobs):

```python
def _update_progress(db, job_id: str, processed: int, total: int) -> None:
    db.execute(
        "UPDATE import_jobs SET processed_rows = :p WHERE id = :id",
        {"p": processed, "id": job_id}
    )
    db.commit()
```

## Retry Strategy

| Error Type                        | Action                                               |
| --------------------------------- | ---------------------------------------------------- |
| Transient (network, DB lock)      | `self.retry(exc=exc, countdown=exponential_backoff)` |
| Permanent (bad data, logic error) | Mark job `FAILED`, log, re-raise                     |
| Worker crash (OOM, SIGKILL)       | `acks_late=True` handles automatic re-queue          |

Exponential backoff formula: `60 * (2 ** self.request.retries)` → 60s, 120s, 240s

## DB Sessions Inside Tasks

Celery tasks run outside the FastAPI async context — use **sync** SQLAlchemy sessions:

```python
# CORRECT
with get_sync_session() as db:
    rows = db.execute(select(Employee)).scalars().all()

# WRONG — never use async sessions inside Celery tasks
async with get_async_session() as db: ...
```

## Enqueueing Tasks

```python
# Fire and forget
my_task.delay(job_id)

# With eta / countdown
my_task.apply_async(args=[job_id], countdown=300)

# From inside another async route (FastAPI)
my_task.delay(str(job_id))   # always pass UUID as str
```

## Scheduled Tasks (Celery Beat)

Define in `app/celery.py`:

```python
app.conf.beat_schedule = {
    "workday-sync-daily": {
        "task": "app.tasks.workday_sync.sync_all_employees",
        "schedule": crontab(hour=2, minute=0),   # 02:00 daily
    },
}
```

## Checklist Before Merging a New Task

- [ ] `acks_late=True` and `reject_on_worker_lost=True` set
- [ ] `max_retries` defined — never infinite retries
- [ ] Permanent errors write to DB status column before re-raising
- [ ] No `async` functions in task body — sync only
- [ ] Task idempotent: safe to run twice with same args
- [ ] UUID args passed as `str`, not `UUID` object
