---
name: add-endpoint
description: "Scaffold a new FastAPI API endpoint with router, Pydantic schemas, SQLAlchemy model (if needed), and Celery task (if async). Use when adding a new REST endpoint to apps/backend/."
argument-hint: "Describe the endpoint: e.g. 'POST /api/certifications to add a certification to an employee profile'"
---

# Add FastAPI Endpoint

Scaffolds a complete, consistent FastAPI endpoint following project conventions.

## When to Use

- Adding any new REST endpoint to `apps/backend/`
- Need router + schema + optional model + optional Celery task
- Want to ensure RBAC, audit logging, and cache invalidation are not missed

## Procedure

### 1. Clarify Requirements

Ask (or infer from context):

- HTTP method + path
- Request body fields and types
- Response shape
- Which roles may call this endpoint
- Is processing async (needs Celery task)?
- Does it modify employee skills? (→ requires cache bust + audit log)

### 2. Create / Update Router

File: `apps/backend/app/routers/<resource>.py`

```python
from fastapi import APIRouter, Depends, HTTPException
from app.database import get_db
from app.schemas.<resource> import <Request>Schema, <Response>Schema
from app.models.<resource> import <Model>
from app.dependencies import get_current_user

router = APIRouter(prefix="/api/<resource>", tags=["<resource>"])

@router.post("/", response_model=<Response>Schema, status_code=201)
async def create_<resource>(
    payload: <Request>Schema,
    db: AsyncSession = Depends(get_db),
    current_user = Depends(get_current_user),
):
    # 1. RBAC check
    # 2. Business logic
    # 3. Audit log if editing another employee's data
    # 4. Invalidate Redis cache if skills changed
    # 5. Enqueue Celery task if async work needed
    ...
```

Register router in `apps/backend/app/main.py`:

```python
from app.routers.<resource> import router as <resource>_router
app.include_router(<resource>_router)
```

### 3. Create Pydantic Schemas

File: `apps/backend/app/schemas/<resource>.py`

```python
from pydantic import BaseModel, UUID4
from datetime import datetime

class <Resource>Base(BaseModel):
    ...

class <Resource>Create(<Resource>Base):
    ...

class <Resource>Response(<Resource>Base):
    id: UUID4
    created_at: datetime
    model_config = {"from_attributes": True}
```

### 4. Add SQLAlchemy Model (if new table)

File: `apps/backend/app/models/<resource>.py`

```python
from sqlalchemy import Column, String, ForeignKey
from sqlalchemy.dialects.postgresql import UUID
import uuid
from app.database import Base

class <Resource>(Base):
    __tablename__ = "<resources>"
    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    ...
```

Then generate migration:

```bash
docker compose exec backend alembic revision --autogenerate -m "add_<resource>_table"
docker compose exec backend alembic upgrade head
```

### 5. Add Celery Task (if async processing needed)

File: `apps/backend/app/tasks/<resource>_task.py`

```python
from app.celery import celery_app

@celery_app.task
def process_<resource>(record_id: str):
    ...
```

### 6. Checklist Before Finishing

- [ ] RBAC check in route handler
- [ ] `skill_audit_log` + notification queued if editing another user's skills
- [ ] `redis.delete(f"profile:{id}")` + `redis.delete_pattern("search:*")` if skills changed
- [ ] Schema has `model_config = {"from_attributes": True}` for ORM responses
- [ ] Router registered in `main.py`
- [ ] Alembic migration created if new model added
- [ ] For export endpoints: scope filtered by role server-side (HR → all, Manager → team, Employee → self); never stream large exports synchronously
