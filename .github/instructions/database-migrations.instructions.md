---
description: "Use when writing Alembic database migrations, modifying the DB schema, adding columns, creating tables, or backfilling data. Covers naming conventions, async SQLAlchemy 2.0 patterns, auto-detect limitations, and safe rollback procedures."
applyTo: "apps/backend/**"
---

# Database Migration Conventions

## Generating Migrations

```bash
# Auto-generate from model changes
docker compose exec backend alembic revision --autogenerate -m "add_<column>_to_<table>"

# Empty migration (for manual data migrations)
docker compose exec backend alembic revision -m "backfill_<description>"

# Apply
docker compose exec backend alembic upgrade head

# Rollback one step
docker compose exec backend alembic downgrade -1
```

## Naming Convention

```text
<verb>_<column_or_thing>_<to/from/in>_<table>

Examples:
  add_profile_embedding_to_employees
  create_export_jobs_table
  add_hnsw_index_to_employees
  backfill_skills_text_from_employee_skills
  drop_legacy_tags_column_from_skills
```

## Migration File Template

```python
"""add_export_jobs_table

Revision ID: xxxx
Revises: yyyy
Create Date: ...
"""
from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects.postgresql import UUID, JSONB

def upgrade() -> None:
    op.create_table(
        "export_jobs",
        sa.Column("id", UUID(as_uuid=True), primary_key=True,
                  server_default=sa.text("gen_random_uuid()")),
        sa.Column("status", sa.String(20), nullable=False, server_default="PENDING"),
        sa.Column("created_by", UUID(as_uuid=True),
                  sa.ForeignKey("employees.id"), nullable=True),
        sa.Column("created_at", sa.TIMESTAMPTZ, server_default=sa.func.now()),
    )

def downgrade() -> None:
    op.drop_table("export_jobs")
```

## Rules

- **Always implement `downgrade()`** — never leave it empty
- **Never drop a column in the same migration as the code removal** — two-phase: deprecate first, drop in next release
- **pgvector columns**: use `pgvector.sqlalchemy.Vector` type, not raw `sa.String`
- **UUID PKs**: always `server_default=sa.text("gen_random_uuid()")` — not Python-side
- **Timestamps**: always `TIMESTAMPTZ`, never `TIMESTAMP`
- **Auto-detect limitations**: Alembic does NOT detect index changes, custom types, or `CHECK` constraints — add these manually in the migration

## Data Migrations (Backfills)

```python
def upgrade() -> None:
    # Always batch large backfills — never UPDATE without WHERE
    op.execute("""
        UPDATE employees
        SET skills_text = (
            SELECT string_agg(s.name, ' ')
            FROM employee_skills es
            JOIN skills s ON s.id = es.skill_id
            WHERE es.employee_id = employees.id
        )
        WHERE skills_text IS NULL
    """)

def downgrade() -> None:
    op.execute("UPDATE employees SET skills_text = NULL")
```

## Testing a Migration Locally

```bash
# Apply
docker compose exec backend alembic upgrade head

# Verify
docker compose exec backend alembic current

# Test rollback
docker compose exec backend alembic downgrade -1
docker compose exec backend alembic upgrade head
```
