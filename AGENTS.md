# Workforce Platform — Agent Instructions

> Full specification: [README.md](./README.md)

## Project Overview

Cloud-hosted, fully Dockerized workforce platform for 1,400 employees — **Pillar 01: Skill Matrix & Competency Management**.

**Stack:** FastAPI · Next.js · PostgreSQL 16 + pgvector · Redis · Celery · sentence-transformers · Docker Compose · Alembic

## Team

| Engineer | Role | Owns |
| ---------- | ------ | ------ |
| `DusanEngIt` | Tech Lead | Architecture decisions, cross-team coordination, code review, delivery oversight |
| `nemanjaninkovic-1` | Backend Engineer | Auth, Workday sync, DB schema/migrations, Docker, all FastAPI routes, skill CRUD, Excel import/export, notifications |
| `engveselin` | AI / Search Engineer | Embedding model, pgvector search, hybrid re-ranking, caching |
| `ilija-radonjic` | Frontend Engineer | Next.js UI, search UI, profile pages, import/export wizard |

## Repository Layout

```text
workforce-platform/
├── docker-compose.yml
├── docker-compose.override.yml   # local dev hot-reload
├── .env.example
├── apps/
│   ├── backend/                  # FastAPI — nemanjaninkovic-1
│   ├── frontend/                 # Next.js (TypeScript) — ilija-radonjic
│   └── ai-service/               # Embeddings + search — engveselin
├── nginx/nginx.conf
└── db/migrations/                # Alembic
```

## Essential Commands

```bash
docker compose up -d                                             # start all services
docker compose exec backend alembic upgrade head                 # run DB migrations
docker compose exec backend python scripts/seed_taxonomy.py      # seed skill taxonomy
docker compose exec ai-service python rebuild_embeddings.py      # rebuild all profile vectors
celery -A app.celery worker --loglevel=info                      # local Celery worker
```

## Architecture Decisions (do not change without discussion)

| Decision | Rationale |
| ---------- | ----------- |
| **pgvector over Qdrant** | Reuses Postgres; same speed at 1,400 profiles with HNSW index; no extra container |
| **paraphrase-multilingual-MiniLM-L12-v2** | Handles Serbian, English, Italian, Albanian + 46 other languages on CPU; 420MB RAM; Apache 2.0; ~10ms |
| **Hybrid search score** | `0.7 × cosine_similarity + 0.3 × ts_rank` |
| **Redis caching** | search TTL 5min; profile TTL 1h; taxonomy TTL 24h |
| **Celery for async** | Excel import (batches of 100); Workday sync at 02:00; profile re-embedding; Excel export |
| **UUID primary keys** | All tables use `gen_random_uuid()` |
| **Soft-delete only** | `is_active = FALSE` — never hard-delete employees |

## Roles & Permissions

| Role | Can Edit | Export Scope |
| ------ | ---------- | -------------- |
| `EMPLOYEE` | Own profile only | Own profile |
| `LINE_MANAGER` | Subordinates (notifies employee); create custom matrices; trigger ad-hoc review cycles | Team only |
| `TECH_LEAD` | Team members (notifies employee); create custom matrices; trigger ad-hoc review cycles | Team only |
| `HR_COORDINATOR` | All profiles, taxonomy, bulk import + export; **create/edit/delete custom roles** | Full workforce |
| `GENERAL_MANAGEMENT` | Read-only dashboards | None |
| `CUSTOM_ROLE` | Configurable per role (view/edit scope, export, matrices, review cycles) | Configurable |

Custom roles are defined by HR Coordinator via `POST /api/roles/custom`. Permissions stored as JSONB. Effective permission = union of built-in role + any assigned custom roles. See README §3.2.

## File-Specific Instructions

- Backend conventions → [.github/instructions/backend.instructions.md](.github/instructions/backend.instructions.md)
- Frontend conventions → [.github/instructions/frontend.instructions.md](.github/instructions/frontend.instructions.md)
- AI service conventions → [.github/instructions/ai-service.instructions.md](.github/instructions/ai-service.instructions.md)
- DB migrations → [.github/instructions/database-migrations.instructions.md](.github/instructions/database-migrations.instructions.md)
- Celery tasks → [.github/instructions/celery-tasks.instructions.md](.github/instructions/celery-tasks.instructions.md)
- Docker services → [.github/instructions/docker-services.instructions.md](.github/instructions/docker-services.instructions.md)
- Testing → [.github/instructions/testing.instructions.md](.github/instructions/testing.instructions.md)

## Agents

| Agent | File | Use When |
| ----- | ---- | -------- |
| `backend-focus` | [.github/agents/backend-focus.agent.md](.github/agents/backend-focus.agent.md) | Writing routes, models, tasks in `apps/backend/` |
| `frontend-focus` | [.github/agents/frontend-focus.agent.md](.github/agents/frontend-focus.agent.md) | Writing components, pages, API client in `apps/frontend/` |
| `ai-service-focus` | [.github/agents/ai-service-focus.agent.md](.github/agents/ai-service-focus.agent.md) | Embedding, search, hybrid scoring in `apps/ai-service/` |
| `search-debug` | [.github/agents/search-debug.agent.md](.github/agents/search-debug.agent.md) | Read-only: diagnose wrong/missing search results |
| `code-reviewer` | [.github/agents/code-reviewer.agent.md](.github/agents/code-reviewer.agent.md) | Read-only: audit code for RBAC, audit log, cache, soft-delete, async |

## Open Questions

See [Section 17 of the spec](./README.md) — Workday API type, brand colors, and SMTP config are unresolved before implementation.
