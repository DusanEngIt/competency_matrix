---
description: "Use when writing tests for FastAPI routes, SQLAlchemy models, Celery tasks, or service functions in apps/backend/. Covers async pytest patterns, DB rollback fixtures, Redis mocking, Celery task testing, and RBAC test helpers."
applyTo: "apps/backend/tests/**"
---

# Backend Testing Conventions

## Test Setup

```bash
# Run all tests
docker compose exec backend pytest

# Run specific file
docker compose exec backend pytest tests/routers/test_employees.py -v

# Run with coverage
docker compose exec backend pytest --cov=app --cov-report=term-missing
```

## Async Test Fixture Pattern

```python
import pytest
from httpx import AsyncClient
from sqlalchemy.ext.asyncio import AsyncSession
from app.main import app
from app.database import get_db

@pytest.fixture
async def db_session():
    """Each test gets its own transaction, rolled back after."""
    async with engine.begin() as conn:
        await conn.execute(text("SET session_replication_role = replica"))
        async with AsyncSession(bind=conn) as session:
            yield session
            await session.rollback()   # never commits — DB stays clean

@pytest.fixture
async def client(db_session):
    app.dependency_overrides[get_db] = lambda: db_session
    async with AsyncClient(app=app, base_url="http://test") as c:
        yield c
    app.dependency_overrides.clear()
```

## RBAC Test Helper

```python
import pytest
from app.auth import create_access_token

def auth_header(role: str, employee_id: str) -> dict:
    token = create_access_token({"sub": employee_id, "role": role})
    return {"Authorization": f"Bearer {token}"}

# Usage
async def test_hr_can_export(client, employee_factory):
    emp = await employee_factory(role="HR_COORDINATOR")
    resp = await client.post(
        "/api/export",
        headers=auth_header("HR_COORDINATOR", str(emp.id))
    )
    assert resp.status_code == 202

async def test_employee_cannot_export_others(client, employee_factory):
    emp = await employee_factory(role="EMPLOYEE")
    resp = await client.post(
        "/api/export",
        headers=auth_header("EMPLOYEE", str(emp.id))
    )
    assert resp.status_code in (200, 202)   # own profile only — not 403
```

## Mocking External Services

```python
from unittest.mock import AsyncMock, patch

# Mock AI service (httpx call)
@pytest.fixture
def mock_ai_service(monkeypatch):
    monkeypatch.setattr(
        "app.services.ai_client.get_embedding",
        AsyncMock(return_value=[0.1] * 384)
    )

# Mock Redis
@pytest.fixture
def mock_redis(monkeypatch):
    monkeypatch.setattr("app.cache.redis", AsyncMock())
```

## Celery Task Testing

```python
# Test task logic directly (no broker needed)
from app.tasks.import_task import process_import

def test_import_task_success(db_session):
    process_import(job_id="test-uuid")   # call directly, not .delay()
    # assert DB state
```

## Rules

- Never use production DB — always use transaction-scoped rollback fixtures
- Always test all RBAC boundaries (own data, subordinate data, unauthorized access)
- Mock `ai_client`, `redis`, and `workday` at the service layer — never at the HTTP level
- Celery tasks: test logic directly, never via `.delay()` in unit tests
- Every new endpoint needs at minimum: happy path + 403 for unauthorized role
