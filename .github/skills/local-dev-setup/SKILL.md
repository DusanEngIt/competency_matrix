---
name: local-dev-setup
description: "Bootstrap a complete local development environment for the Workforce Platform from scratch. Use when setting up for the first time, recovering from a broken environment, onboarding a new engineer, or needing to reset the dev stack to a clean state."
argument-hint: "Describe your starting point: fresh clone, broken environment, specific service failing, etc."
---

# Local Dev Setup

Bootstrap the complete local dev environment step by step.

## Prerequisites

- Docker + Docker Compose installed (v24+)
- 16GB RAM recommended (AI model needs ~420MB + other services)
- Git access to the repo

## Step 1 — Clone and Configure

```bash
git clone https://github.com/DusanEngIt/competency_matrix.git workforce-platform
cd workforce-platform

# Copy env template and fill in required values
cp .env.example .env
```

Required `.env` values before starting:

```bash
POSTGRES_PASSWORD=<choose a strong password>
JWT_SECRET=<random 32+ char string, e.g.: openssl rand -hex 32>
WORKDAY_API_URL=      # leave blank until credentials available (Section 17)
WORKDAY_CLIENT_ID=    # leave blank until available
WORKDAY_CLIENT_SECRET=# leave blank until available
```

## Step 2 — Start All Services

```bash
docker compose up -d
```

Wait ~60 seconds on first run (AI model downloads ~420MB).

Check all services are healthy:

```bash
docker compose ps
# All services should show "running" or "healthy"
```

## Step 3 — Initialize the Database

```bash
# Run Alembic migrations
docker compose exec backend alembic upgrade head

# Seed the skill taxonomy
docker compose exec backend python scripts/seed_taxonomy.py
```

## Step 4 — Build Profile Embeddings

```bash
docker compose exec ai-service python rebuild_embeddings.py
# Takes ~50ms per employee — fast for a fresh empty DB
```

## Step 5 — Verify Services

```bash
# Backend API
curl -s http://localhost:8000/health

# AI service
curl -s http://localhost:5000/health

# Frontend
open http://localhost:3000   # or curl -s -o /dev/null -w "%{http_code}" http://localhost:3000

# Postgres
docker compose exec postgres psql -U postgres -c "SELECT COUNT(*) FROM employees"

# Redis
docker compose exec redis redis-cli ping
```

## Step 6 — Enable Hot-Reload (optional)

`docker-compose.override.yml` is already configured for hot-reload. No changes needed — edits to `apps/backend/` and `apps/frontend/` will auto-reload.

## Troubleshooting

### AI model download hangs

```bash
docker compose logs ai-service --tail=20
# If stuck: docker compose restart ai-service
# Model is cached in the ai_models volume — only downloads once
```

### Port already in use

```bash
# Find what's using port 8000
lsof -i :8000
# Add port override in docker-compose.override.yml
```

### Backend fails to start (DB not ready)

```bash
docker compose logs backend --tail=20
# Usually postgres is still initializing — wait 10s and retry:
docker compose restart backend
```

### Alembic "target database is not up to date"

```bash
docker compose exec backend alembic stamp head   # reset to current
docker compose exec backend alembic upgrade head
```

### Full reset (⚠️ destroys all local data)

```bash
docker compose down -v   # removes volumes including DB data
docker compose up -d
# Then redo Steps 3–5
```
