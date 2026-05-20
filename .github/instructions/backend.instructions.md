---
description: "Use when writing FastAPI routes, SQLAlchemy models, Pydantic schemas, Celery tasks, Redis caching, or any backend Python code in apps/backend/. Covers async patterns, RBAC, audit logging, cache invalidation, and API conventions."
applyTo: "apps/backend/**"
---

# Backend Conventions

## Async SQLAlchemy 2.0

- Always `async def` + `await` for all DB calls — never use sync sessions
- Engine/session in `app/database.py`; inject via `Depends(get_db)` in routes
- Use `select()`, `insert()`, `update()` from `sqlalchemy` — not ORM `.query()`

## Team Split

| Engineer            | Owns                                                                                                          |
| ------------------- | ------------------------------------------------------------------------------------------------------------- |
| `nemanjaninkovic-1` | All backend: auth, Workday sync, DB schema/migrations, Docker, skill CRUD, Excel import/export, notifications |

## Project Structure

```text
app/
├── routers/          # auth, employees, skills, search, import, export, reviews, notifications
├── models/           # SQLAlchemy ORM models
├── schemas/          # Pydantic v2 request/response schemas
├── services/         # Business logic (workday, excel, notifications, ai_client)
└── tasks/            # import_task, export_task, workday_sync, embed_profile, review_reminders
```

## RBAC Enforcement

Check role inside route handler via decoded JWT — do not trust client-supplied role claims:

```python
# Roles: EMPLOYEE | LINE_MANAGER | TECH_LEAD | HR_COORDINATOR | GENERAL_MANAGEMENT
if current_user.role not in ("HR_COORDINATOR",):
    raise HTTPException(status_code=403, detail="Forbidden")
```

Manager editing a subordinate's skill **must** create a `skill_audit_log` entry and queue a notification task — never skip this.

## Audit Log Pattern (every manager/HR edit)

```python
# 1. Write audit log row
# 2. Celery: tasks.notifications.send_skill_change_notification.delay(...)
```

## Cache Invalidation

```python
# After any profile skill change:
await redis.delete(f"profile:{employee_id}")
await redis.delete_pattern("search:*")    # bust all search caches
# After taxonomy change:
await redis.delete_pattern("taxonomy:*")
```

## Excel Export

- Endpoint: `POST /api/export` → enqueues `export_task`; `GET /api/export/{job_id}/status`; `GET /api/export/{job_id}/download`
- Scope enforced by role: `HR_COORDINATOR` → full workforce; `LINE_MANAGER` + `TECH_LEAD` → subordinates/team only; `EMPLOYEE` → own profile only
- Generated server-side with `openpyxl`; never stream large exports synchronously — always use Celery
- Column mapping uses English field names: `First Name`, `Last Name`, `Department`, `Position`, `Technology`, `Category`, `Level (1-5)`, `Years of Exp`, `Notes`

## Settings

- All config via `pydantic-settings` in `app/config.py`
- **Never** hardcode secrets, URLs, or credentials — always `settings.<field>`

## Libraries in Use

`fastapi` · `uvicorn[standard]` · `sqlalchemy[asyncio]==2.0.*` · `asyncpg` · `alembic`
`pydantic-settings` · `python-jose[cryptography]` · `passlib[bcrypt]` · `redis[asyncio]`
`celery[redis]` · `httpx` · `openpyxl` · `pandas` · `python-multipart` · `pgvector`
