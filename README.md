# Workforce Platform — Full Requirements Specification

## Pillar 01: Skill Matrix & Competency Management

> **ENG Software Lab · Workforce Platform Workshop**
> Version 5.0 | FastAPI · Local AI · pgvector · Fully Dockerized

---

## Table of Contents

1. [Team & Responsibilities](#1-team--responsibilities)
2. [Project Overview](#2-project-overview)
3. [Permission Model](#3-permission-model)
4. [Skill Taxonomy](#4-skill-taxonomy)
5. [Excel Import](#5-excel-import)
6. [Functional Requirements](#6-functional-requirements)
7. [Search Architecture](#7-search-architecture--fast--low-cost)
8. [Technical Architecture](#8-technical-architecture)
9. [Backend — FastAPI](#9-backend--fastapi)
10. [AI Service](#10-ai-service)
11. [Frontend — Next.js](#11-frontend--nextjs)
12. [Data Model](#12-data-model)
13. [Workday Integration](#13-workday-integration)
14. [Deployment](#14-deployment)
15. [Non-Functional Requirements](#15-non-functional-requirements)
16. [Repository Structure](#16-repository-structure)
17. [Open Questions](#17-open-questions)
18. [Milestones](#18-milestones)

---

## 1. Team & Responsibilities

| Engineer | Role | Owns |
| ---------- | ------ | ------ |
| **DusanEngIt** | 🏗️ Tech Lead | Architecture decisions, cross-team coordination, code review, delivery oversight |
| **nemanjaninkovic-1** | ⚙️ Backend Engineer | Auth, Workday sync, DB schema/migrations, Docker setup, all FastAPI routes, skill CRUD, Excel import/export, notifications |
| **engveselin** | 🤖 AI / Search Engineer | Embedding model, pgvector search, semantic pipeline, caching |
| **ilija-radonjic** | 🎨 Frontend Engineer | Next.js UI, company branding, search UI, profile pages, import wizard |

---

## 2. Project Overview

### 2.1 What We Are Building

A **cloud-hosted, fully Dockerized workforce platform** for 1,400 employees that:

- Centralizes every employee's hard and soft skills in one place
- Allows each employee to manage their own profile
- Enables fast AI-powered semantic search across all profiles
- Runs on cloud infrastructure — no SaaS, no external AI API costs
- Integrates with Workday as the HR system of record
- Supports bulk data import from existing Excel files

### 2.2 The Six Pillars (Today's Scope: Pillar 01)

| # | Pillar | Status |
| --- | -------- | -------- |
| **01** | **Skill Matrix & Competency Management** | ✅ Today's scope |
| 02 | Workforce Management & Resource Allocation | 🔜 Future |
| 03 | Demand Management & Staffing Tracking | 🔜 Future |
| 04 | Search & Discovery | 🔜 Future |
| 05 | CV Generation & Profile Export | 🔜 Future |
| 06 | Reports & Workforce Analytics | 🔜 Future |

### 2.3 Core Principles

- **Local AI first** — all inference runs inside Docker, zero external API calls
- **pgvector over Qdrant** — one less container, same speed, zero extra cost
- **Redis caching** — sub-20ms repeat searches
- **No subscriptions** — one VM, fully open-source stack
- **Turnkey** — `docker compose up` and it works

---

## 3. Permission Model

```text
┌──────────────────────┬───────────────┬──────────────────────────────────┐
│ ROLE                 │ VIEW          │ EDIT                             │
├──────────────────────┼───────────────┼──────────────────────────────────┤
│ Employee (all 1,400) │ All profiles  │ Own profile only                 │
├──────────────────────┼───────────────┼──────────────────────────────────┤
│ Line Manager         │ All profiles  │ Subordinates                     │
│                      │               │ ⚠️ Notifies employee on change   │
├──────────────────────┼───────────────┼──────────────────────────────────┤
│ Tech Lead            │ All profiles  │ Team members                     │
│                      │               │ ⚠️ Notifies employee on change   │
├──────────────────────┼───────────────┼──────────────────────────────────┤
│ HR Coordinator       │ All + analytics│ Taxonomy, bulk import, reviews  │
├──────────────────────┼───────────────┼──────────────────────────────────┤
│ General Management   │ Dashboards    │ No editing                       │
└──────────────────────┴───────────────┴──────────────────────────────────┘
```

### Manager Edit — Notification Flow

```text
Manager saves edit
      │
      ▼
SkillAuditLog entry created
      │
      ▼
Notification triggered
  ├── In-app alert (seen on next login)
  └── Email (sent immediately)
        Content: skill changed · old → new · by whom · timestamp
        Action:  "Dispute this change" → opens HR ticket
```

---

## 4. Skill Taxonomy

### 4.1 Hard Skills vs. Soft Skills

```text
HARD SKILLS
├── Programming Languages     Java, Python, TypeScript, Go, C#...
├── Frameworks & Libraries    React, Angular, Spring Boot, FastAPI...
├── Cloud & DevOps            AWS, Azure, Docker, Kubernetes, Terraform...
├── Databases                 PostgreSQL, MongoDB, Redis, Elasticsearch...
├── Architecture              Microservices, Event-driven, DDD, REST...
├── Domain Knowledge          Banking, Telecom, Insurance, ERP...
└── Certifications            AWS SAA, PMP, Scrum Master, CKAD...

SOFT SKILLS
├── Communication
├── Leadership & Team Management
├── Problem Solving & Critical Thinking
├── Adaptability & Learning Agility
├── Client Facing / Stakeholder Management
├── Mentoring & Knowledge Transfer
├── Estimation                      — realistic estimation of time, effort, and complexity
├── Requirements Understanding      — clear grasp of goals, context, and expectations
├── Work Breakdown (WBS)            — decomposing work into clear, manageable, logical steps
├── Risk Awareness                  — timely identification and addressing of risks
├── Ownership & Accountability      — taking responsibility from start to finish
├── Stakeholder Communication       — clear, timely, and audience-appropriate communication
└── Delivery Reliability            — consistent delivery on agreed timelines and quality
```

### 4.2 Proficiency Scale

| Level | Label | Description |
| ------- | ------- | ------------- |
| 1 | Beginner | Basic understanding; frequent support needed; works on clearly defined, simple tasks |
| 2 | Intermediate | Independent on simpler tasks; needs support on complex or less defined tasks |
| 3 | Experienced | Stable and reliable in daily work; independently manages medium-complexity tasks; consciously considers risks and dependencies |
| 4 | Advanced | Very confident and proactive; works on complex, multi-layered tasks; able to provide realistic estimation, work breakdown, and planning even under uncertainty |
| 5 | Master | Expert level; works on highly complex or strategic topics; mentors others and actively improves delivery processes |

### 4.3 Adding New Skills

```text
Employee types unknown skill
      ├── AI suggests closest taxonomy match (local embeddings)
      │         ├── Match accepted → uses existing entry
      │         └── No match → submitted as proposal → HR approves
      │
HR Admin Panel → add / edit / deprecate skills manually
      │
Excel Import  → new skill names auto-proposed for HR review
```

---

## 5. Excel Import

### 5.1 Import Flow

```text
STEP 1 — UPLOAD
  HR or Employee uploads .xlsx / .xls / .csv (max 50MB)

STEP 2 — COLUMN MAPPING WIZARD
  System reads row 1 headers
  User maps Excel columns → platform fields

  Excel Column        →   Platform Field
  ──────────────────────────────────────────────
  "First Name"        →   employee.first_name
  "Last Name"         →   employee.last_name
  "Email"             →   employee.email  ← unique key
  "Department"        →   employee.department
  "Position"          →   employee.title
  "Technology"        →   skill.name
  "Category"          →   skill.category (HARD/SOFT)
  "Level (1-5)"       →   proficiency_level
  "Years of Exp"      →   years_of_experience
  "Notes"             →   notes

  Mapping template saved for future reuse

STEP 3 — VALIDATION
  ✅ Required fields present (email, skill, proficiency)
  ✅ Proficiency is 1–5 (text levels auto-converted: Beginner→1, Intermediate→2, Experienced→3, Advanced→4, Master→5;
           legacy maps: Junior→2, Mid→3, Senior→4 also supported)
  ✅ Employee matched by email
  ⚠️  Unknown skill → AI maps to closest taxonomy entry
  ❌  Unresolvable skill → flagged, excluded, HR notified
  ❌  Duplicate row   → deduplicated, user notified

  Result: GREEN (ok) / YELLOW (auto-mapped) / RED (error)

STEP 4 — PREVIEW & CONFIRM
  Paginated table with row status indicators
  User can exclude specific rows before committing

STEP 5 — BACKGROUND PROCESSING (Celery)
  Batches of 100 rows
  Progress bar in UI (polling /import/{job_id}/status)
  Summary report: ✅ X ok  ⚠️ Y mapped  ❌ Z failed
  Downloadable error report for failed rows
```

---

## 6. Functional Requirements

### 6.1 Employee Profile

```text
├── Basic Info (synced from Workday)
│   ├── Full name, email, photo, department, title, seniority, manager
│
├── Hard Skills
│   ├── Skill name · Proficiency (1–5) · Years of experience · Notes
│
├── Soft Skills
│   ├── Skill name · Proficiency (1–5)
│
├── Certifications
│   ├── Name · Issuing body · Expiry date (with expiry warning)
│
└── Audit History
    └── Full changelog: who changed what and when
```

### 6.2 Skill Self-Service

- Autocomplete from taxonomy while typing
- Free-text → AI maps to closest taxonomy match in real time (local)
- Proficiency slider (1–5) with label shown
- Free-text notes per skill
- Changes saved instantly (optimistic UI)

### 6.3 Excel Export

```text
SCOPE
  HR Coordinator  → full workforce export (all employees, all skills)
  Line Manager    → team export (subordinates only)
  Employee        → own profile export

FORMAT: .xlsx   (generated server-side via openpyxl)

COLUMNS EXPORTED
  First Name · Last Name · Email · Department · Position · Seniority
  Skill Name · Category (HARD/SOFT) · Proficiency (1–5) · Years of Exp · Notes
  Last Reviewed At · Last Reviewed By

FILTERS (optional)
  Department, Category, Min Proficiency Level

EXPORT FLOW
  User clicks "Export" → POST /api/export → Celery task → .xlsx generated
  Download link returned when ready  (or streamed directly for small sets)
```

### 6.4 Review Cycles

```text
Semi-annual  → automated reminders at T-14 days, manager dashboard
Annual       → employee updates + manager sign-off required
Project-based → triggered manually by HR or Manager
```

---

## 7. Search Architecture — Fast & Low Cost

### 7.1 Why pgvector Instead of Qdrant

| | Qdrant (separate container) | pgvector (PostgreSQL extension) |
| -- | -- | -- |
| Extra container | ✅ Yes — extra RAM, extra ops | ❌ No — reuses existing DB |
| Speed at 1,400 profiles | Fast | **Equally fast** (HNSW index) |
| Cost | Free but heavier | **Free + lighter** |
| Ops complexity | Two DBs to manage | **One DB** |
| Backup | Separate | **Included in Postgres backup** |

> **Decision: pgvector.** For 1,400 employees, pgvector with an HNSW index is indistinguishable in speed from Qdrant, with zero added complexity.

### 7.2 Semantic Search Pipeline

```text
User query: "cloud architecture, team leadership"
      │
      ▼
┌─────────────────────────────────────────────────────┐
│  ai-service (FastAPI container)                     │
│                                                     │
│  1. Normalize + detect language (SR/EN/mixed)       │
│                                                     │
│  2. Generate embedding                              │
│     Model: paraphrase-multilingual-MiniLM-L12-v2    │
│     → 384-dim vector · ~10ms on CPU · 420MB RAM     │
│     → Apache 2.0 · no API cost · no GPU needed      │
│                                                     │
│  3. pgvector cosine similarity search               │
│     SELECT employees ORDER BY                       │
│     embedding <=> query_vector LIMIT 50             │
│     (HNSW index — ~5ms for 1,400 profiles)          │
│                                                     │
│  4. Hybrid re-ranking                               │
│     score = 0.7 × vector_score                      │
│           + 0.3 × PostgreSQL full-text (ts_rank)    │
│                                                     │
│  5. Cache result in Redis                           │
│     Key: search:{sha256(query+filters)}  TTL: 5min  │
└─────────────────────────────────────────────────────┘
      │
      ▼
Ranked employee list returned in < 100ms (cold)
                                  < 10ms  (cached)
```

### 7.3 pgvector Setup

```sql
-- Enable extension (runs once on DB init)
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS pg_trgm;   -- for BM25 hybrid

-- Add vector column to employee_skills aggregate view
ALTER TABLE employees
  ADD COLUMN profile_embedding vector(384);

-- HNSW index for fast ANN search
CREATE INDEX ON employees
  USING hnsw (profile_embedding vector_cosine_ops)
  WITH (m = 16, ef_construction = 64);

-- Full-text search index for hybrid re-ranking
CREATE INDEX ON employees
  USING GIN (to_tsvector('english', skills_text));

-- Profile vector = weighted mean of skill embeddings
-- Rebuilt async on every profile save (Celery task)
-- Rebuild time: ~50ms per profile
```

### 7.4 Redis Caching

```text
KEY                                          TTL
────────────────────────────────────────────────────────
search:{sha256(query+filters)}               5 min
profile:{employee_id}                        1 hour
taxonomy:skills:{category}                  24 hours
notifications:unread:{employee_id}           5 min

INVALIDATION
  Employee updates skill  → DEL profile:{id}, DEL search:*
  Manager updates skill   → DEL profile:{id}, queue notification
  Bulk import done        → DEL search:*
  Taxonomy updated        → DEL taxonomy:*
```

---

## 8. Technical Architecture

### 8.1 Docker Services

```text
┌──────────────────────────────────────────────────────────────────┐
│                       DOCKER HOST (Cloud VM)                     │
│                                                                  │
│  nginx:443       reverse proxy + SSL termination                 │
│  frontend:3000   Next.js                                         │
│  backend:8000    FastAPI                                         │
│  ai-service:5000 FastAPI + sentence-transformers                 │
│  postgres:5432   PostgreSQL 16 + pgvector + pg_trgm              │
│  redis:6379      Cache layer                                     │
│  celery          Background jobs (import, sync, notifications)   │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
                            │
               Workday API (daily sync at 02:00)
```

### 8.2 docker-compose.yml

```yaml
version: "3.9"

services:

  nginx:
    image: nginx:alpine
    ports: ["80:80", "443:443"]
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./certs:/etc/ssl/certs
    depends_on: [frontend, backend]

  frontend:
    build: ./apps/frontend
    environment:
      - NEXT_PUBLIC_API_URL=/api
    depends_on: [backend]

  backend:
    build: ./apps/backend
    environment:
      - DATABASE_URL=postgresql+asyncpg://postgres:${POSTGRES_PASSWORD}@postgres:5432/workforce
      - REDIS_URL=redis://redis:6379
      - AI_SERVICE_URL=http://ai-service:5000
      - CELERY_BROKER_URL=redis://redis:6379/1
      - WORKDAY_API_URL=${WORKDAY_API_URL}
      - WORKDAY_CLIENT_ID=${WORKDAY_CLIENT_ID}
      - WORKDAY_CLIENT_SECRET=${WORKDAY_CLIENT_SECRET}
      - JWT_SECRET=${JWT_SECRET}
    depends_on: [postgres, redis, ai-service]

  ai-service:
    build: ./apps/ai-service
    environment:
      - DATABASE_URL=postgresql+asyncpg://postgres:${POSTGRES_PASSWORD}@postgres:5432/workforce
      - MODEL_NAME=paraphrase-multilingual-MiniLM-L12-v2
    volumes:
      - ai_models:/app/models      # model weights cached on disk
    depends_on: [postgres]

  celery:
    build: ./apps/backend
    command: celery -A app.celery worker --loglevel=info
    environment:
      - DATABASE_URL=postgresql+asyncpg://postgres:${POSTGRES_PASSWORD}@postgres:5432/workforce
      - REDIS_URL=redis://redis:6379
      - CELERY_BROKER_URL=redis://redis:6379/1
      - AI_SERVICE_URL=http://ai-service:5000
    depends_on: [postgres, redis]

  postgres:
    image: pgvector/pgvector:pg16   # official pgvector image
    environment:
      - POSTGRES_DB=workforce
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory 512mb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
  ai_models:
```

---

## 9. Backend — FastAPI

### 9.1 Project Structure

```text
apps/backend/
├── Dockerfile
├── requirements.txt
├── alembic.ini
└── app/
    ├── main.py                   # FastAPI app, middleware, routers
    ├── config.py                 # pydantic-settings
    ├── database.py               # async SQLAlchemy engine
    ├── cache.py                  # Redis helpers
    ├── celery.py                 # Celery instance
    │
    ├── routers/
    │   ├── auth.py               # POST /auth/login, /auth/refresh, /auth/sso
    │   ├── employees.py          # GET/PUT /employees, /employees/{id}
    │   ├── skills.py             # taxonomy CRUD
    │   ├── employee_skills.py    # skill entries per employee
    │   ├── search.py             # GET /search?q=...
    │   ├── import.py             # Excel upload + processing
    │   ├── reviews.py            # review cycles
    │   └── notifications.py      # user inbox
    │
    ├── models/                   # SQLAlchemy ORM models
    ├── schemas/                  # Pydantic request/response schemas
    │
    ├── services/
    │   ├── workday.py            # Workday sync
    │   ├── excel.py              # openpyxl + pandas import parser
    │   ├── notifications.py      # email + in-app
    │   └── ai_client.py          # httpx client → ai-service
    │
    └── tasks/                    # Celery background tasks
        ├── import_task.py        # Excel batch processing
        ├── workday_sync.py       # daily cron
        ├── embed_profile.py      # rebuild profile vector after edit
        └── review_reminders.py   # scheduled notifications
```

### 9.2 API Endpoints

```text
AUTH
POST   /api/auth/login
POST   /api/auth/sso
POST   /api/auth/refresh

EMPLOYEES
GET    /api/employees                      paginated, filterable
GET    /api/employees/{id}
PUT    /api/employees/{id}

SKILLS (taxonomy)
GET    /api/skills?category=HARD|SOFT
POST   /api/skills                         HR only
PUT    /api/skills/{id}
DELETE /api/skills/{id}

EMPLOYEE SKILLS
GET    /api/employees/{id}/skills
POST   /api/employees/{id}/skills
PUT    /api/employees/{id}/skills/{sid}
DELETE /api/employees/{id}/skills/{sid}

SEARCH
GET    /api/search?q=...&category=...&min_level=...&department=...

IMPORT
POST   /api/import/upload
GET    /api/import/{job_id}/preview
POST   /api/import/{job_id}/confirm
GET    /api/import/{job_id}/status

REVIEWS
GET    /api/reviews
POST   /api/reviews                        HR only
GET    /api/reviews/{id}/status

NOTIFICATIONS
GET    /api/notifications
PATCH  /api/notifications/{id}/read

EXPORT
POST   /api/export                        HR / Manager / Employee (scoped)
GET    /api/export/{job_id}/status
GET    /api/export/{job_id}/download
```

### 9.3 Libraries

```text
fastapi · uvicorn[standard]
sqlalchemy[asyncio]==2.0.* · asyncpg · alembic
pydantic-settings
python-jose[cryptography]       # JWT
passlib[bcrypt]
redis[asyncio]
celery[redis]
httpx                           # calls ai-service
openpyxl · pandas               # Excel import + export
python-multipart                # file upload
pgvector                        # SQLAlchemy vector type
```

---

## 10. AI Service

### 10.1 Project Structure

```text
apps/ai-service/
├── Dockerfile
├── requirements.txt
├── main.py            # FastAPI: POST /embed, POST /search
├── embeddings.py      # sentence-transformers model wrapper
├── search.py          # pgvector query + hybrid re-ranking
└── models/            # cached model weights (Docker volume)
```

### 10.2 Model Choice

| Model | RAM | Languages | CPU Speed | License |
| ------- | ----- | ----------- | ----------- | --------- |
| **paraphrase-multilingual-MiniLM-L12-v2** ✅ | 420MB | 50+ (incl. Serbian) | ~10ms | Apache 2.0 |
| all-MiniLM-L6-v2 | 80MB | English only | ~5ms | Apache 2.0 |
| multilingual-e5-large | 1.1GB | 100+ | ~40ms | MIT |

**Chosen model handles Serbian + English natively, runs on CPU, free forever.**

### 10.3 Libraries

```text
fastapi · uvicorn
sentence-transformers
asyncpg · sqlalchemy[asyncio]
pgvector
rank-bm25                       # hybrid keyword re-ranking
```

---

## 11. Frontend — Next.js

### 11.1 Project Structure

```text
apps/frontend/
├── Dockerfile
├── package.json
└── src/
    ├── app/
    │   ├── layout.tsx
    │   ├── page.tsx                  # dashboard
    │   ├── search/page.tsx           # semantic search
    │   ├── profile/[id]/page.tsx     # view profile
    │   ├── profile/edit/page.tsx     # self-service editing
    │   ├── import/page.tsx           # Excel import wizard
    │   ├── export/page.tsx           # Excel export
    │   └── admin/
    │       ├── taxonomy/page.tsx
    │       └── reviews/page.tsx
    │
    ├── components/
    │   ├── ui/
    │   │   ├── Button.tsx
    │   │   ├── Badge.tsx             # proficiency level badges
    │   │   ├── SkillCard.tsx
    │   │   ├── ProfileCard.tsx
    │   │   └── SearchBar.tsx
    │   ├── import/
    │   │   ├── FileUpload.tsx
    │   │   ├── ColumnMapper.tsx
    │   │   ├── ValidationTable.tsx
    │   │   └── ImportProgress.tsx
    │   └── layout/
    │       ├── Navbar.tsx
    │       └── Sidebar.tsx
    │
    ├── styles/
    │   └── globals.css              # brand colors as CSS variables
    └── lib/
        ├── api.ts                   # typed API client
        ├── auth.ts                  # NextAuth.js
        └── types.ts
```

### 11.2 Brand Colors

```css
/* Replace with actual company hex values */
:root {
  --color-primary:         #YOUR_PRIMARY;
  --color-primary-dark:    #YOUR_DARK;
  --color-secondary:       #YOUR_SECONDARY;
  --color-background:      #FAFAFA;
  --color-surface:         #FFFFFF;
  --color-text-primary:    #1A1A2E;
  --color-text-secondary:  #6B7280;
  --color-success:         #10B981;
  --color-warning:         #F59E0B;
  --color-error:           #EF4444;
}
```

---

## 12. Data Model

```sql
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS pg_trgm;

CREATE TABLE employees (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  workday_id        VARCHAR(50) UNIQUE,
  first_name        VARCHAR(100) NOT NULL,
  last_name         VARCHAR(100) NOT NULL,
  email             VARCHAR(200) UNIQUE NOT NULL,
  department        VARCHAR(100),
  title             VARCHAR(100),
  seniority_level   VARCHAR(50),
  manager_id        UUID REFERENCES employees(id),
  role              VARCHAR(20) DEFAULT 'EMPLOYEE',
  profile_embedding vector(384),          -- pgvector: mean of skill embeddings
  skills_text       TEXT,                 -- denormalized for full-text search
  is_active         BOOLEAN DEFAULT TRUE,
  created_at        TIMESTAMPTZ DEFAULT NOW(),
  updated_at        TIMESTAMPTZ DEFAULT NOW()
);

-- HNSW index for vector similarity search (~5ms at 1,400 rows)
CREATE INDEX ON employees
  USING hnsw (profile_embedding vector_cosine_ops)
  WITH (m = 16, ef_construction = 64);

-- Full-text index for hybrid re-ranking
CREATE INDEX ON employees
  USING GIN (to_tsvector('english', coalesce(skills_text, '')));

CREATE TABLE skills (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name             VARCHAR(200) NOT NULL,
  category         VARCHAR(10) NOT NULL,   -- HARD | SOFT
  description      TEXT,
  embedding        vector(384),            -- skill name embedding
  is_active        BOOLEAN DEFAULT TRUE,
  created_by       UUID REFERENCES employees(id),
  approved_by      UUID REFERENCES employees(id),
  created_at       TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE employee_skills (
  id                   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  employee_id          UUID NOT NULL REFERENCES employees(id),
  skill_id             UUID NOT NULL REFERENCES skills(id),
  proficiency_level    INT CHECK (proficiency_level BETWEEN 1 AND 5),
  years_of_experience  NUMERIC(4,1),
  notes                TEXT,
  last_reviewed_at     TIMESTAMPTZ,
  last_reviewed_by     UUID REFERENCES employees(id),
  created_at           TIMESTAMPTZ DEFAULT NOW(),
  updated_at           TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(employee_id, skill_id)
);

CREATE TABLE skill_audit_log (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  employee_id       UUID REFERENCES employees(id),
  changed_by        UUID REFERENCES employees(id),
  change_type       VARCHAR(10),           -- CREATE | UPDATE | DELETE
  old_value         JSONB,
  new_value         JSONB,
  notification_sent BOOLEAN DEFAULT FALSE,
  created_at        TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE review_cycles (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name        VARCHAR(200),
  type        VARCHAR(20),                 -- SEMI_ANNUAL | ANNUAL | PROJECT
  start_date  DATE,
  end_date    DATE,
  status      VARCHAR(20) DEFAULT 'ACTIVE',
  created_by  UUID REFERENCES employees(id),
  created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE employee_review_status (
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  review_cycle_id       UUID REFERENCES review_cycles(id),
  employee_id           UUID REFERENCES employees(id),
  status                VARCHAR(20) DEFAULT 'PENDING',
  completed_at          TIMESTAMPTZ,
  manager_validated_at  TIMESTAMPTZ,
  UNIQUE(review_cycle_id, employee_id)
);

CREATE TABLE import_jobs (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  filename        VARCHAR(255),
  status          VARCHAR(20) DEFAULT 'PENDING',
  total_rows      INT,
  processed_rows  INT DEFAULT 0,
  success_rows    INT DEFAULT 0,
  error_rows      INT DEFAULT 0,
  column_mapping  JSONB,
  error_report    JSONB,
  created_by      UUID REFERENCES employees(id),
  created_at      TIMESTAMPTZ DEFAULT NOW(),
  completed_at    TIMESTAMPTZ
);
```

---

## 13. Workday Integration

```text
DIRECTION: Workday → Platform only (one-way, Pillar 01)
SCHEDULE:  Daily cron at 02:00 (Celery beat inside backend container)
AUTH:      Workday OAuth2 / RAAS REST API

SYNC EVENTS
  All active employees  → upsert employees table (key: workday_id)
  Org chart             → update manager_id relationships
  New hire detected     → create profile + send welcome email
  Termination detected  → set is_active = FALSE (data retained)
```

---

## 14. Deployment

### 14.1 VM Sizing

| Provider | VM | vCPU | RAM | Cost/mo |
| ---------- | ----- | ------ | ----- | --------- |
| Hetzner (EU) | CX31 | 2 | 8GB | ~€12 |
| AWS | t3.large | 2 | 8GB | ~$55 |
| **Azure B4ms** *(recommended)* | | **4** | **16GB** | **~$80** |

> 16GB recommended: AI model (~420MB) + PostgreSQL + Redis + all app containers comfortably fit.

### 14.2 Startup

```bash
# Install Docker on Ubuntu 22.04
curl -fsSL https://get.docker.com | sh

# Clone & configure
git clone https://github.com/org/workforce-platform.git
cd workforce-platform
cp .env.example .env        # fill in passwords + Workday credentials

# Launch
docker compose up -d

# Initialize
docker compose exec backend alembic upgrade head
docker compose exec backend python scripts/seed_taxonomy.py
docker compose exec ai-service python rebuild_embeddings.py

# Platform live at https://your-domain.com
```

### 14.3 Backup

```bash
# Daily PostgreSQL backup (cron on VM)
0 3 * * * docker compose exec -T postgres \
  pg_dump -U postgres workforce | \
  gzip > /backups/workforce_$(date +%Y%m%d).sql.gz

# Keep 30 days
find /backups -name "*.sql.gz" -mtime +30 -delete

# Redis → cache only, no backup needed
# pgvector data → included in Postgres backup automatically
```

---

## 15. Non-Functional Requirements

| Requirement | Target |
| ------------- | -------- |
| Search (cold) | < 100ms |
| Search (cached) | < 10ms |
| AI embedding | < 10ms on CPU |
| Page load | < 2s |
| Concurrent users | 200+ |
| Uptime | 99.5% (`restart: always` on all containers) |
| Data residency | Stays on EU VM — never sent to external AI |
| GDPR | Audit trail, right to access, right to delete |
| Security | TLS 1.3, JWT, RBAC, secrets in env |
| Accessibility | WCAG 2.1 AA |
| Mobile | Responsive web |
| Language | Serbian + English |

---

## 16. Repository Structure

```text
workforce-platform/
├── docker-compose.yml
├── docker-compose.override.yml      # local dev hot-reload
├── .env.example
├── apps/
│   ├── backend/                     # FastAPI — nemanjaninkovic-1
│   ├── frontend/                    # Next.js — ilija-radonjic
│   └── ai-service/                  # Embeddings + search — engveselin
├── nginx/nginx.conf
├── db/migrations/
└── README.md                        # this file
```

---

## 17. Open Questions

| # | Question | Owner | Priority |
| --- | ---------- | ------- | ---------- |
| 1 | Workday API type (REST/RAAS/SOAP) + credentials? | nemanjaninkovic-1 | 🔴 High |
| 2 | Company brand colors (hex)? | ilija-radonjic | 🔴 High |
| 3 | SSO provider — Azure AD, Okta, or LDAP? | nemanjaninkovic-1 | 🔴 High |
| 4 | Sample anonymized Excel export from HR? | nemanjaninkovic-1 | 🔴 High |
| 5 | Cloud provider preference / existing VM? | nemanjaninkovic-1 | 🟡 Medium |
| 6 | Proficiency self-reported only, or manager must validate? | nemanjaninkovic-1 | 🟡 Medium |
| 7 | UI language — Serbian, English, or both? | ilija-radonjic | 🟡 Medium |
| 8 | Dispute flow when employee disagrees with manager edit? | nemanjaninkovic-1 | 🟡 Medium |
| 9 | SMTP server available for email notifications? | nemanjaninkovic-1 | 🟡 Medium |
| 10 | How is proficiency stored in current Excel? (numbers or text?) | nemanjaninkovic-1 | 🟡 Medium |

---

## 18. Milestones

| Week | Milestone | Owner(s) |
| ------ | ----------- | ---------- |
| 1 | Docker compose running, DB schema + pgvector setup, migrations | nemanjaninkovic-1 |
| 1 | AI service running, embedding model loaded, /embed endpoint live | engveselin |
| 2 | FastAPI auth endpoints (login, SSO, refresh) | nemanjaninkovic-1 |
| 2 | Employee + skill CRUD endpoints | nemanjaninkovic-1 |
| 2 | Next.js scaffolded, branding applied, routing set up | ilija-radonjic |
| 3 | Excel import wizard end-to-end | nemanjaninkovic-1 + ilija-radonjic |
| 3 | Semantic search endpoint + Redis caching | engveselin + nemanjaninkovic-1 |
| 4 | Search UI, profile view, skill editing UI | ilija-radonjic |
| 4 | Workday sync service (daily cron) | nemanjaninkovic-1 |
| 5 | Notification system (in-app + email) | nemanjaninkovic-1 |
| 5 | Review cycle management | nemanjaninkovic-1 + ilija-radonjic |
| 6 | Cloud deploy, SSL, smoke testing, UAT with HR | Full team |

---

*ENG Software Lab · Workforce Platform · Pillar 01*
*FastAPI · Next.js · PostgreSQL + pgvector · Redis · Local AI · Docker*
*Version 5.0*
