---
description: "Use when working with Docker Compose services, rebuilding containers, debugging container health, viewing logs, managing volumes, or troubleshooting local dev environment issues across any service."
applyTo: "**/{docker-compose*.yml,Dockerfile}"
---

# Docker Services Conventions

## Essential Commands

```bash
# Start all services
docker compose up -d

# Rebuild a single service after code change
docker compose up -d --build backend

# View live logs for a service (with filtering)
docker compose logs -f backend
docker compose logs -f celery | grep -E "ERROR|WARNING|task"

# Restart a single service without rebuild
docker compose restart celery

# Stop everything and remove containers (keeps volumes)
docker compose down

# Full reset — removes volumes too (⚠️ destroys DB data)
docker compose down -v
```

## Service Health Checks

```bash
# Check all container statuses
docker compose ps

# Check a specific service's health
docker inspect competency_matrix-backend-1 | jq '.[0].State.Health'

# Backend API alive
curl -s http://localhost:8000/health | jq

# AI service alive
curl -s http://localhost:5000/health | jq

# Postgres reachable
docker compose exec postgres psql -U postgres -c "SELECT 1"

# Redis reachable
docker compose exec redis redis-cli ping
```

## Environment Variables

- Defaults: `docker-compose.yml`
- Local overrides: `docker-compose.override.yml` — never commit secrets here
- Source of truth: `.env` file (copy from `.env.example`)
- Override precedence: `.env` < `docker-compose.yml` < `docker-compose.override.yml` < shell env

## Debugging OOM / Container Crashes

```bash
# Check exit code (137 = OOM kill)
docker compose ps -a

# Check system memory
free -h
docker stats --no-stream

# AI service needs ~420MB for model — ensure host has ≥ 2GB free
```

## Hot-Reload (Local Dev)

`docker-compose.override.yml` mounts source code and enables hot-reload:

```yaml
services:
  backend:
    volumes:
      - ./apps/backend:/app
    command: uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
  frontend:
    volumes:
      - ./apps/frontend:/app
    command: npm run dev
```

Do not use `--reload` in production.

## Scaling Celery Workers

```bash
# Run 3 Celery workers (e.g., during bulk import)
docker compose up -d --scale celery=3
```

## Port Conflicts

| Service    | Default Port |
| ---------- | ------------ |
| nginx      | 80, 443      |
| frontend   | 3000         |
| backend    | 8000         |
| ai-service | 5000         |
| postgres   | 5432         |
| redis      | 6379         |

Change ports in `docker-compose.override.yml` only — never in `docker-compose.yml`.
