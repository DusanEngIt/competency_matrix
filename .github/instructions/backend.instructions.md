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

## Authentication — Azure Active Directory (MSAL + OIDC)

- SSO provider is **Azure AD**. Use `msal` (Python) on the backend.
- Backend validates Azure AD JWTs using the JWKS endpoint:
  `https://login.microsoftonline.com/{AZURE_TENANT_ID}/discovery/v2.0/keys`
- Required `settings` fields: `AZURE_TENANT_ID`, `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`
- Platform role is derived from the Azure AD group claim in the token. Map AD group IDs to roles in `app/auth/role_mapping.py`.
- Username/password fallback is only active when `DEV_AUTH=true` (local dev only — never in production).
- Token validation dependency lives in `app/auth/dependencies.py`; inject as `Depends(get_current_user)` in all protected routes.

## RBAC Enforcement

Check role inside route handler via decoded JWT — do not trust client-supplied role claims:

```python
# Built-in roles: EMPLOYEE | LINE_MANAGER | TECH_LEAD | HR_COORDINATOR | GENERAL_MANAGEMENT
# LINE_MANAGER / TECH_LEAD view scope:
#   - own subordinates/team: full profile + matrix accessible
#   - other employees: name + department only (skill data must be excluded from response)
if current_user.role not in ("HR_COORDINATOR",):
    raise HTTPException(status_code=403, detail="Forbidden")
```

**Profile response scope enforcement:** When `LINE_MANAGER` or `TECH_LEAD` requests a profile for a non-subordinate, return only `{id, full_name, department, title}` — strip all skill data server-side. Never rely on frontend to hide this.

**Custom roles:** In addition to the built-in role, employees may have custom roles assigned via `employee_custom_roles`. To check a custom permission:

```python
# Effective permissions = built-in role permissions UNION custom role permissions
has_perm = await check_effective_permission(db, current_user.id, "can_export", scope="DEPARTMENT")
if not has_perm:
    raise HTTPException(status_code=403, detail="Forbidden")
```

- `check_effective_permission` lives in `app/auth/permissions.py`.
- Custom role permissions are stored in `custom_roles.permissions JSONB`; never exceed `HR_COORDINATOR` level.
- Only `HR_COORDINATOR` can create/edit/delete custom roles (`POST/PUT/DELETE /api/roles/custom`).

## Proficiency Validation Status

`employee_skills.validation_status` has three values: `PENDING` · `CONFIRMED` · `REJECTED`.

- Employee saves a skill → status = `PENDING`, manager notified.
- Manager confirms → status = `CONFIRMED`, employee notified.
- Manager edits to a different value → status = `CONFIRMED` immediately, old value replaced, employee notified.
- `PENDING` skills are included in search results with a visual indicator; `REJECTED` skills are excluded.
- Only `LINE_MANAGER` / `TECH_LEAD` can confirm skills for their own subordinates / team members.

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
`pydantic-settings` · `msal` · `python-jose[cryptography]` · `passlib[bcrypt]` · `redis[asyncio]`
`celery[redis]` · `httpx` · `openpyxl` · `pandas` · `python-multipart` · `pgvector`
