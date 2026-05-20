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
18. [Educated Guesses — Pending Decisions](#18-educated-guesses--pending-decisions)
19. [Milestones](#19-milestones)

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

### 3.1 Built-in Roles

Five system roles are pre-defined and cannot be deleted:

```text
┌──────────────────────┬───────────────┬──────────────────────────────────┐
│ ROLE                 │ VIEW          │ EDIT                             │
├──────────────────────┼───────────────┼──────────────────────────────────┤
│ Employee (all 1,400) │ Hierarchy     │ Own profile only                 │
│                      │ (name, dept,  │                                  │
│                      │  title, org   │                                  │
│                      │  chart only — │                                  │
│                      │  no matrices) │                                  │
├──────────────────────┼───────────────┼──────────────────────────────────┤
│ Line Manager         │ Hierarchy     │ Subordinates                     │
│                      │ (name, dept,  │ ⚠️ Notifies employee on change   │
│                      │  title, org   │ ✦ Create custom skill matrices   │
│                      │  chart) +     │ ✦ Propose new skills for matrix  │
│                      │ full matrix   │ ✦ Trigger ad-hoc review cycles   │
│                      │ for own team  │   (before/after project, outside │
│                      │               │   standard semi-annual/annual)   │
├──────────────────────┼───────────────┼──────────────────────────────────┤
│ Tech Lead            │ Hierarchy     │ Team members                     │
│                      │ (name, dept,  │ ⚠️ Notifies employee on change   │
│                      │  title, org   │ ✦ Create custom skill matrices   │
│                      │  chart) +     │ ✦ Propose new skills for matrix  │
│                      │ full matrix   │ ✦ Trigger ad-hoc review cycles   │
│                      │ for own team  │                                  │
├──────────────────────┼───────────────┼──────────────────────────────────┤
│ HR Coordinator       │ All + analytics│ Taxonomy, bulk import, reviews  │
│                      │               │ ✦ Create / edit / delete custom  │
│                      │               │   roles (see §3.2)               │
├──────────────────────┼───────────────┼──────────────────────────────────┤
│ General Management   │ All profiles  │ No editing                       │
│                      │ + analytics   │                                  │
└──────────────────────┴───────────────┴──────────────────────────────────┘
```

### 3.2 Custom Roles

HR Coordinators can define additional roles beyond the five built-in ones to accommodate org-specific needs (e.g., "Project Manager", "Scrum Master", "Department Head").

**Configurable permissions per custom role:**

| Permission | Description |
| ---------- | ----------- |
| `view_scope` | `OWN` · `TEAM` · `DEPARTMENT` · `ALL` |
| `edit_scope` | `NONE` · `OWN` · `TEAM` · `DEPARTMENT` |
| `can_edit_subordinates` | Boolean — whether edit triggers employee notification |
| `can_create_matrices` | Boolean — create custom skill matrices |
| `can_trigger_review_cycles` | Boolean — start ad-hoc review cycles |
| `can_export` | `NONE` · `OWN` · `TEAM` · `DEPARTMENT` · `ALL` |
| `can_manage_taxonomy` | Boolean — propose/approve skills (HR-level) |

**Rules:**

- Custom roles are stored in the `custom_roles` DB table with a `permissions JSONB` column.
- A custom role's permissions cannot exceed those of `HR_COORDINATOR` (server-side enforcement).
- Employees can be assigned a custom role in addition to (not instead of) their built-in role. The effective permission is the union of both.
- Custom roles are visible in the role assignment UI only to `HR_COORDINATOR`.
- Built-in roles (`EMPLOYEE`, `LINE_MANAGER`, `TECH_LEAD`, `HR_COORDINATOR`, `GENERAL_MANAGEMENT`) are stored as an enum in code and cannot be modified or deleted.

**DB schema:**

```sql
CREATE TABLE custom_roles (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        VARCHAR(100) NOT NULL UNIQUE,
    description TEXT,
    permissions JSONB NOT NULL DEFAULT '{}',
    created_by  UUID REFERENCES employees(id),
    created_at  TIMESTAMPTZ DEFAULT now(),
    is_active   BOOLEAN DEFAULT TRUE
);

CREATE TABLE employee_custom_roles (
    employee_id     UUID REFERENCES employees(id) ON DELETE CASCADE,
    custom_role_id  UUID REFERENCES custom_roles(id) ON DELETE CASCADE,
    assigned_by     UUID REFERENCES employees(id),
    assigned_at     TIMESTAMPTZ DEFAULT now(),
    PRIMARY KEY (employee_id, custom_role_id)
);
```

**API endpoints** (owner: `nemanjaninkovic-1`):

```text
GET    /api/roles/custom                    # list all custom roles
POST   /api/roles/custom                    # create custom role (HR only)
GET    /api/roles/custom/{role_id}          # get role details
PUT    /api/roles/custom/{role_id}          # update role (HR only)
DELETE /api/roles/custom/{role_id}          # soft-delete role (HR only)
POST   /api/roles/custom/{role_id}/assign   # assign role to employee (HR only)
DELETE /api/roles/custom/{role_id}/assign/{employee_id}  # unassign
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
├── Estimation                      — realistic assessment of time, effort, and complexity, aligned with task scope
├── Requirements Understanding      — clear understanding of goals, context, and expectations
├── Work Breakdown (WBS)            — breaking down work into clear, manageable, and logical steps
├── Risk Awareness                  — timely identification and addressing of risks
├── Independence & Ownership        — taking responsibility from start to finish
├── Stakeholder Communication       — clear, timely, and tailored communication
└── Delivery Reliability            — consistent delivery within agreed timelines and quality
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

> **Source of truth:** The master employee skill data is maintained by HR in an **Excel file on SharePoint**. The platform import wizard ingests this file. For Pillar 01, HR downloads the file from SharePoint and uploads it via the wizard. Direct SharePoint integration via Microsoft Graph API is a future consideration.

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

### 5.2 Import Flow Diagram

```mermaid
%%{init: {
  "theme": "base",
  "themeVariables": {
    "primaryColor": "#c51b88",
    "primaryTextColor": "#fcfcfc",
    "primaryBorderColor": "#8c1e74",
    "lineColor": "#323551",
    "secondaryColor": "#f4d1e8",
    "tertiaryColor": "#323551",
    "tertiaryTextColor": "#fcfcfc",
    "background": "#fcfcfc",
    "nodeBorder": "#8c1e74",
    "clusterBkg": "#f4d1e8",
    "titleColor": "#161721",
    "edgeLabelBackground": "#f4d1e8",
    "fontFamily": "ui-sans-serif, system-ui, sans-serif"
  }
}}%%
flowchart TD
    A([HR downloads Excel\nfrom SharePoint]) --> B[Upload .xlsx via wizard\nPOST /api/import/upload]
    B --> C[Column Mapping\nMap Excel headers → platform fields]
    C --> D[Validation]

    D --> D1{Required fields\npresent?}
    D1 -- No --> E1[❌ Row rejected\nHR notified]
    D1 -- Yes --> D2{Proficiency\n1–5?}
    D2 -- No --> D2a[Auto-convert text label\nBeginner→1 … Master→5]
    D2a --> D3
    D2 -- Yes --> D3{Employee\nmatched by email?}
    D3 -- No --> E2[❌ Row rejected]
    D3 -- Yes --> D4{Skill in\ntaxonomy?}
    D4 -- Yes --> G[✅ Row valid]
    D4 -- No --> D5[AI maps to closest\ntaxonomy entry]
    D5 -- Match found --> G2[⚠️ Auto-mapped row]
    D5 -- No match --> E3[❌ Flagged — HR notified\nto approve new skill]

    G --> H[Preview & Confirm\nPaginated table with status]
    G2 --> H
    H --> I[User confirms\nPOST /api/import/confirm]
    I --> J[Celery task enqueued\nimport_employees_task]

    J --> K[Process in batches of 100]
    K --> L[Upsert employees + skills\nin PostgreSQL]
    L --> M[Trigger embed_profile\nCelery task per employee]
    M --> N[(pgvector updated\n384-dim embedding)]
    K --> O[Poll status\nGET /api/import/job_id/status]
    O --> P([Summary report\n✅ ok / ⚠️ mapped / ❌ failed])
```

---

## 6. Functional Requirements

### 6.0 Role-Based UI Flows

#### Employee — Application Experience

```text
LOGIN (Azure AD SSO)
      │
      ▼
HOME / DASHBOARD
  ├── Notifications badge — pending review requests, manager edits
  ├── "Fill in your matrix" banner — shown during active review window
  └── Quick link → own profile

MY PROFILE
  ├── Basic info (read-only, synced from Workday)
  ├── Hard Skills tab
  │   ├── List of skills with proficiency level + years of exp
  │   └── Editable at any time; saved as PENDING until manager confirms
  ├── Soft Skills tab
  │   └── Same as Hard Skills
  ├── Audit History tab
  │   └── Full changelog: who changed what and when
  └── Export own profile → .xlsx

MATRIX FILL FLOW (triggered by review cycle or manager request)
  ├── Notification received (in-app + email)
  ├── Employee opens assigned matrix
  ├── For each skill: selects proficiency (1–5) + optional notes
  ├── Saves (progress auto-saved as draft)
  └── Submits → manager notified of completion

ACTIVE REVIEW WINDOWS
  ├── Semi-annual — HR-triggered, platform-wide
  ├── Annual — HR-triggered, manager sign-off required
  └── Ad-hoc — triggered by Line Manager / Tech Lead at any time
```

#### Line Manager — Application Experience

```text
LOGIN (Azure AD SSO)
      │
      ▼
HOME / DASHBOARD
  ├── Notifications badge
  ├── Pending matrix completions from subordinates
  └── Review cycle status summary for team

MY TEAM (subordinates only — full access)
  ├── List of direct reports with completion status
  ├── Click employee → full profile + skill matrix
  ├── Edit any skill → employee notified automatically
  └── Export team matrix → .xlsx

OTHER EMPLOYEES (non-subordinates — limited view)
  ├── Visible in employee directory and search results
  ├── Name, department, position shown
  └── Skill matrix is HIDDEN — no access

MATRIX MANAGEMENT
  ├── "Request matrix fill" — send to one or all subordinates at any time
  ├── Create new matrix:
  │   ├── Option A: pick from existing default matrix templates
  │   ├── Option B: select existing skills from taxonomy
  │   └── Option C: propose new skills (submitted to HR for approval)
  ├── Assign matrix to one or more subordinates
  ├── Set deadline
  ├── Track completion progress per employee
  └── View results when employees submit

REVIEW CYCLES
  ├── Participate in HR-triggered semi-annual / annual cycles
  └── Trigger ad-hoc cycle at any time (project start/end, role change)
```

#### Tech Lead — Application Experience

```text
LOGIN (Azure AD SSO)
      │
      ▼
HOME / DASHBOARD
  ├── Notifications badge
  ├── Pending matrix completions from team members
  └── Review cycle status summary for team

MY TEAM (team members — full access)
  ├── List of team members with completion status
  ├── Click employee → full profile + skill matrix
  ├── Edit any skill → employee notified automatically
  └── Export team matrix → .xlsx

OTHER EMPLOYEES (outside team — limited view)
  ├── Visible in employee directory and search results
  ├── Name, department, position shown
  └── Skill matrix is HIDDEN — no access

MATRIX MANAGEMENT
  ├── Same capabilities as Line Manager
  └── Propose new skills for taxonomy (submitted to HR for approval)

REVIEW CYCLES
  ├── Participate in HR-triggered semi-annual / annual cycles
  └── Trigger ad-hoc cycle at any time
```

#### HR Coordinator — Application Experience

```text
LOGIN (Azure AD SSO)
      │
      ▼
HOME / DASHBOARD
  ├── Notifications badge
  ├── Platform-wide review cycle status (% completed, overdue)
  ├── Pending taxonomy proposals from managers
  └── Recent import/export job statuses

EMPLOYEE DIRECTORY (full access — all 1,400 employees)
  ├── Search bar + filters (department, position, skills, proficiency)
  ├── Click any employee → full profile + skill matrix
  ├── Edit any employee's skills directly
  └── Bulk actions: trigger review cycle, export selected profiles

EXCEL IMPORT WIZARD
  ├── Upload .xlsx from SharePoint
  ├── Column mapping screen (auto-detected, manually adjustable)
  ├── Validation summary (errors, warnings, skipped rows)
  ├── Preview changes before confirm
  ├── Confirm → Celery job runs in background
  └── Job status page → downloadable import report on completion

EXCEL EXPORT
  ├── Export scope: full workforce / by department / by custom filter
  ├── Format: same structure as import template
  └── Download link generated by Celery job (email notification on completion)

TAXONOMY MANAGEMENT
  ├── View all skills (active + deprecated)
  ├── Review pending skill proposals from imports and managers
  │   ├── Approve → skill added to taxonomy
  │   ├── Reject → proposer notified
  │   └── Merge → map to existing skill
  ├── Add / edit / deprecate skills directly
  └── Taxonomy cache invalidated on any change

REVIEW CYCLE MANAGEMENT
  ├── Create semi-annual or annual cycle
  │   ├── Select scope: all employees or specific departments
  │   ├── Set deadline
  │   └── Launch → notifications sent to all employees and their managers
  ├── Monitor completion (table: employee / status / last updated)
  ├── Send reminders (individual or bulk)
  ├── Extend deadline
  └── Finalize / close cycle → manager sign-off required for annual

CUSTOM ROLE MANAGEMENT
  ├── Create new custom role (name + description)
  ├── Configure permissions (JSONB: view_scope, can_export, can_create_matrices, etc.)
  ├── Assign custom role to one or more employees
  └── Edit / delete custom roles (ceiling = HR_COORDINATOR permissions)

PLATFORM SETTINGS
  ├── Search minimum match threshold (default 70%, range 50–95%)
  ├── Search page size (default 10, range 5–50)
  └── Changes take effect immediately; search cache invalidated automatically
```

#### General Management — Application Experience

```text
LOGIN (Azure AD SSO)
      │
      ▼
HOME / DASHBOARD
  ├── Org-wide skill heatmap (skill × department, colour = avg proficiency)
  ├── Top skills by headcount
  ├── Skills gap alerts (required vs. actual per department)
  └── Review cycle completion rates (current cycle progress)

EMPLOYEE DIRECTORY (full read-only access)
  ├── Search bar + filters (same as HR Coordinator)
  ├── Click any employee → full profile + skill matrix (read-only)
  └── No edit controls visible

ANALYTICS DASHBOARDS
  ├── Headcount per skill / proficiency level
  ├── Department skill breakdown
  ├── Proficiency distribution (Beginner → Master)
  ├── Skills added / confirmed / pending over time
  └── Export charts as PDF (browser print)

NO EDITING
  └── All forms and edit buttons hidden for this role
```

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
STANDARD CYCLES (HR-managed)
  Semi-annual  → automated reminders at T-14 days, manager dashboard
  Annual       → employee updates + manager sign-off required

AD-HOC CYCLES (Line Manager / Tech Lead)
  Project-based → triggered manually before or after a project
  Custom matrix → manager defines a specific skill set to evaluate
                  for selected team members, outside standard schedule

CUSTOM MATRIX FLOW
  Manager creates matrix
    → Selects or proposes skills (proposed skills go to HR for approval)
    → Assigns matrix to one or more subordinates
    → Sets deadline
    → Employees notified to fill in proficiency for the listed skills
    → Manager sees results when completed
    → Results visible in employee profile audit history
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
User query: "cloud architecture, team leadership" + filters: {department: "Engineering", title: "PM"}
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
│  3. Pre-filter (SQL WHERE before vector search)     │
│     department = ? AND/OR title ILIKE ?             │
│     Reduces candidate set before HNSW step          │
│                                                     │
│  4. pgvector cosine similarity search               │
│     SELECT employees ORDER BY                       │
│     embedding <=> query_vector LIMIT 100            │
│     (HNSW index — ~5ms for 1,400 profiles)          │
│                                                     │
│  5. Hybrid re-ranking + score filter                │
│     score = 0.60 × vector_score                     │
│           + 0.25 × PostgreSQL full-text (ts_rank)   │
│           + 0.15 × recency_score(last_reviewed_at)  │
│  Discard results below min_score threshold (def 0.70)│
│                                                     │
│  6. Paginate — page_size results per page (def 10)  │
│     Return page N via offset / cursor               │
│                                                     │
│  7. Cache result in Redis                           │
│     Key: search:{sha256(query+filters+page)}        │
│     TTL: 5min                                       │
└─────────────────────────────────────────────────────┘
      │
      ▼
Ranked employee list returned in < 100ms (cold)
                                  < 10ms  (cached)
```

### 7.2a Search Flow Diagram

```mermaid
%%{init: {
  "theme": "base",
  "themeVariables": {
    "primaryColor": "#c51b88",
    "primaryTextColor": "#fcfcfc",
    "primaryBorderColor": "#8c1e74",
    "lineColor": "#323551",
    "secondaryColor": "#f4d1e8",
    "tertiaryColor": "#323551",
    "tertiaryTextColor": "#fcfcfc",
    "background": "#fcfcfc",
    "nodeBorder": "#8c1e74",
    "clusterBkg": "#f4d1e8",
    "titleColor": "#161721",
    "edgeLabelBackground": "#f4d1e8",
    "fontFamily": "ui-sans-serif, system-ui, sans-serif"
  }
}}%%
flowchart TD
    A([User enters query + optional filters\ndepartment / title / role type]) --> B{Redis cache hit?\nsearch:sha256 query+filters+page}
    B -- Hit --> Z([Return cached result\n< 10ms])
    B -- Miss --> C[Normalize text\nDetect language SR/EN/IT/AL]

    C --> D[Generate 384-dim embedding\nparaphrase-multilingual-MiniLM-L12-v2\n~10ms on CPU]

    D --> E[Pre-filter candidates\nSQL WHERE department=? AND title ILIKE ?]

    E --> F[pgvector ANN search\nHNSW cosine similarity\nLIMIT 100 candidates · ~5ms]

    F --> G[Hybrid re-ranking\nscore = 0.60 × cosine\n      + 0.25 × ts_rank\n      + 0.15 × recency_score]

    G --> H{score ≥ min_score?}
    H -- No --> I[Discard result]
    H -- Yes --> J[Keep result]

    J --> K[Paginate\npage_size per page]
    K --> L[Cache in Redis\nTTL 5 min]
    L --> M([Return ranked list\nwith stale-profile badge\nif last_reviewed_at > 12 months])
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
    │   ├── export.py             # Excel export (scoped by role)
    │   ├── reviews.py            # review cycles
    │   ├── matrices.py           # custom skill matrices (Manager / Tech Lead)
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

CUSTOM MATRICES
GET    /api/matrices                       Manager / Tech Lead — own matrices
POST   /api/matrices                       Manager / Tech Lead — create matrix
GET    /api/matrices/{id}
PUT    /api/matrices/{id}
DELETE /api/matrices/{id}
POST   /api/matrices/{id}/assign           Assign matrix to employee(s)
GET    /api/matrices/{id}/results          View completion results

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

**Chosen model handles Serbian, English, Italian, Albanian and 46 other languages natively, runs on CPU, free forever.**

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

Derived from [eng.it](https://www.eng.it/en) — source: production CSS.

```css
:root {
  /* ENG Brand — Primary Magenta */
  --color-primary:         #c51b88;
  --color-primary-dark:    #8c1e74;
  --color-primary-light:   #f4d1e8;

  /* ENG Brand — Dark Surfaces */
  --color-dark-navy:       #323551;
  --color-near-black:      #161721;

  /* Neutral */
  --color-background:      #fcfcfc;
  --color-surface:         #ffffff;
  --color-text-primary:    #090909;
  --color-text-secondary:  #9d9d9d;

  /* Semantic */
  --color-success:         #198754;
  --color-warning:         #ffc107;
  --color-error:           #da1e28;
  --color-info:            #323551;
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
| Language | Serbian, English, Italian, Albanian (i18n) |

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

Quick-reference summary:

| # | Question | Owner | Priority | Status |
| --- | ---------- | ------- | ---------- | ------ |
| 1 | Workday API type + credentials | nemanjaninkovic-1 | 🔴 High | ⬜ Open |
| 2 | ~~Company brand colors~~ ✅ **Answered** — ENG brand palette extracted from eng.it | ilija-radonjic | 🔴 High | ✅ Answered |
| 3 | ~~SSO provider~~ ✅ **Answered** — Azure Active Directory (MSAL / OIDC) | nemanjaninkovic-1 | 🔴 High | ✅ Answered |
| 4 | ~~Sample anonymized Excel from HR~~ ✅ **Answered** — Excel on SharePoint, HR is source of truth | nemanjaninkovic-1 | 🔴 High | ✅ Answered |
| 5 | ~~Search ranking~~ ✅ **Answered** — most recent review ranks highest (time-decay applied) | engveselin | 🔴 High | ✅ Answered |
| 6 | ~~Search results — 0 matching skills?~~ ✅ **Answered** — default 70% min score, 10/page; both HR-configurable | engveselin | 🔴 High | ✅ Answered |
| 7 | Pillar 01 MVP scope — what is in vs. deferred? | DusanEngIt | 🔴 High | ⚠️ Guess (§18.3) |
| 8 | ~~Cloud provider~~ ✅ **Answered** — infrastructure provided by ENG | nemanjaninkovic-1 | 🟡 Medium | ✅ Answered |
| 9 | ~~Proficiency validation~~ ✅ **Answered** — self-reported, awaiting manager confirmation | nemanjaninkovic-1 | 🟡 Medium | ✅ Answered |
| 10 | UI language | ilija-radonjic | 🟡 Medium | ✅ Answered |
| 11 | Dispute flow for manager-edited proficiency | nemanjaninkovic-1 | 🟡 Medium | ⬜ Open |
| 12 | SMTP configuration for email notifications | nemanjaninkovic-1 | 🟡 Medium | ⬜ Open |
| 13 | ~~Proficiency format in current Excel files~~ — N/A | nemanjaninkovic-1 | 🟡 Medium | 🚫 N/A |
| 14 | Search filters — department, position, role type (PM etc.) | engveselin | 🔴 High | ✅ Answered |
| 15 | Employee profile photos — source and storage | ilija-radonjic | 🟡 Medium | ⬜ Open |
| 16 | Accessibility and compliance requirements | ilija-radonjic | 🟡 Medium | ⬜ Open |
| 17 | Workday sync failure SLA and alerting | DusanEngIt | 🟡 Medium | ⬜ Open |
| 18 | Data retention policy for audit logs and terminated employees | DusanEngIt | 🟡 Medium | ⬜ Open |

---

### Detailed Breakdown

---

#### Q1 — Workday API Type + Credentials 🔴 High · nemanjaninkovic-1

**What we need to know:**

- Is the Workday integration REST API, RAAS (Report-as-a-Service), or SOAP Web Services?
- Authentication method: OAuth 2.0 client credentials, basic auth, or API key?
- Workday tenant URL, client ID, client secret, and the list of available fields (employee ID, full name, department, manager, job title, hire date, status).
- Any rate limits or quotas on the API?

**Why it matters:**
The sync service is being designed around Workday REST + OAuth 2.0. If the actual integration uses RAAS (CSV over HTTP) or SOAP, the `workday_service.py` client and its auth flow require a full rewrite. This decision gates Week 4 of the timeline.

**Current assumption:** REST API with OAuth 2.0, daily sync at 02:00, one-way (Workday → platform).

---

#### Q2 — Company Brand Colors ✅ Answered · ilija-radonjic

**Answer:** Brand colors extracted from [eng.it](https://www.eng.it/en) CSS.

| Token | Hex | Usage |
| ----- | --- | ----- |
| Primary | `#c51b88` | Buttons, links, active states, highlights |
| Primary Dark | `#8c1e74` | Hover/pressed states on primary |
| Primary Light | `#f4d1e8` | Backgrounds, badges, subtle tints |
| Dark Navy | `#323551` | Headers, sidebar, dark surfaces |
| Near Black | `#161721` | Page backgrounds (dark mode) / deep contrast |
| Text Primary | `#090909` | Body text |
| Background | `#fcfcfc` | Page background |
| Neutral | `#9d9d9d` | Borders, disabled states, secondary text |

**Still needed:** Logo SVG file(s) for light and dark backgrounds. Typography (font family) not identified from CSS — confirm with HR/design team.

---

#### Q3 — SSO Provider ✅ Answered · nemanjaninkovic-1

**Answer:** **Azure Active Directory (Azure AD)** via MSAL + OIDC.

**What still needs to be provided:**

- Azure AD **tenant ID**
- Application (client) **ID** and **client secret** (registered app in Azure AD)
- **Redirect URIs** to register in the Azure AD app (e.g., `https://<host>/auth/callback`)
- **AD group → platform role mapping** (which Azure AD group corresponds to EMPLOYEE, LINE_MANAGER, TECH_LEAD, HR_COORDINATOR, GENERAL_MANAGEMENT)

**Implementation notes:**

- Library: `msal` (Python) on the backend, `next-auth` with Azure AD provider on the frontend.
- Token validation: backend validates Azure AD JWT using JWKS endpoint (`https://login.microsoftonline.com/<tenant>/discovery/v2.0/keys`).
- Username/password fallback retained for local dev only (`DEV_AUTH=true` env flag).
- Profile photos can optionally be pulled from Microsoft Graph API (`/me/photo/$value`) — see Q15.

---

#### Q4 — Sample Anonymized Excel from HR ✅ Answered · nemanjaninkovic-1

**Answer:** The Excel file is stored on **SharePoint** and is the **source of truth** for employee skill data. HR maintains it there and provides it for import into the platform.

**Implications:**

- The import wizard must support a **SharePoint URL / file picker** as an upload source (in addition to local file upload), or HR downloads the file and uploads it manually through the wizard UI.
- The Excel column structure will be confirmed once HR shares the SharePoint file with the team. `nemanjaninkovic-1` must then:
  1. Map real column headers in `excel_import.py`.
  2. Confirm proficiency format and update the normalizer.
  3. Validate against the real file before UAT.
- Because SharePoint is the source of truth, **conflicting edits** (employee updates in the platform vs. HR updating the SharePoint file) must be resolved by policy. See **§18.2** for the educated-guess conflict resolution policy (platform-wins by default, with HR override flag).
- Future consideration: Microsoft Graph API can pull the file directly from SharePoint on a schedule, removing the manual download step (out of scope for Pillar 01).

---

#### Q5 — Search Ranking: Recency of Last Review ✅ Answered · engveselin

**Answer:** The most recently reviewed profile ranks highest. A **time-decay factor** is applied to the hybrid score so that freshly reviewed skills carry more weight than stale ones.

**Updated score formula:**

```python
# recency_score: 1.0 if reviewed within 6 months, decays linearly to 0.5 at 24 months, 0.5 floor beyond
import math
from datetime import datetime, timezone

def recency_score(last_reviewed_at):
    if last_reviewed_at is None:
        return 0.5  # never reviewed — minimum weight
    months_ago = (datetime.now(timezone.utc) - last_reviewed_at).days / 30
    return max(0.5, 1.0 - (months_ago / 48))  # linear decay over 24 months

hybrid_score = (
    0.60 * vector_cosine_similarity
  + 0.25 * ts_rank
  + 0.15 * recency_score(last_reviewed_at)
)
```

**Rules:**

- Profiles reviewed within the last 6 months receive full recency weight.
- Profiles never reviewed get a `0.5` recency floor — they still appear but rank lower.
- A ⚠️ **"Stale profile"** badge appears on search result cards when `last_reviewed_at` is older than 12 months or `NULL`.
- `last_reviewed_at` is indexed: `CREATE INDEX ON employees(last_reviewed_at)`.
- Score weights changed from `0.7 / 0.3` to `0.60 / 0.25 / 0.15` — update `AGENTS.md` architecture decision and `ai-service.instructions.md`.

---

#### Q6 — Search Results ✅ Answered · engveselin

**Answer:**

- **Minimum match threshold: configurable, default 70%** — results with a hybrid score below the threshold are excluded. HR Coordinator can adjust this in platform settings (allowed range: 50%–95%).
- **Page size: configurable, default 10 results per page** — HR Coordinator can change this in platform settings (allowed range: 5–50).
- Zero-match (low-relevance) employees are not shown.

**Implementation:**

```python
# Settings loaded from DB (cached in Redis, TTL 24h):
threshold = settings.search_min_score   # default 0.70
page_size  = settings.search_page_size  # default 10

# After hybrid re-ranking:
results = [r for r in ranked if r.hybrid_score >= threshold]

# Paginate
page = results[offset : offset + page_size]
total_pages = ceil(len(results) / page_size)
```

- `POST /api/search` accepts `page` (default `1`) and returns `{results, page, total_pages, total_count}`.
- Page number is included in the Redis cache key: `search:{sha256(query+filters+threshold+page_size)}:page:{n}`, TTL 5 min.
- Cache is invalidated when HR Coordinator changes either setting.
- Frontend shows "No results above {threshold}% match" when the filtered list is empty.

**Platform settings stored in:** `platform_settings` table — `search_min_score NUMERIC DEFAULT 0.70` and `search_page_size INTEGER DEFAULT 10`. Editable via `PATCH /api/settings/search` (HR_COORDINATOR only).

---

#### Q7 — Pillar 01 MVP Scope 🔴 High · DusanEngIt

**What we need to know:**

- Which of the following features are required for the MVP launch vs. deferred to a later pillar?
  - Workday sync (vs. manual import only for MVP)
  - Review cycles (semi-annual + annual)
  - Custom skill matrices for Line Managers / Tech Leads
  - Email notifications (vs. in-app only for MVP)
  - Excel export
  - GENERAL_MANAGEMENT read-only dashboards
  - Employee self-service profile editing
- Is there a hard go-live date for Pillar 01?

**Why it matters:**
The current spec includes all of the above. If Workday sync or review cycles are deferred, Week 4–5 of the timeline is freed up for hardening core features. Misaligned scope expectations are the #1 risk to the 6-week delivery estimate.

**Current assumption:** See **§18.3** for the educated-guess MVP boundary. DusanEngIt to confirm before Week 2.

---

#### Q8 — Cloud Provider / Existing Infrastructure ✅ Answered · nemanjaninkovic-1

**Answer:** Cloud infrastructure is **provided by ENG** (ENG Software Lab).

**What still needs to be confirmed:**

- Server specs (RAM/CPU/disk) and OS — Azure B4ms (4 vCPU, 16 GB) remains the reference minimum.
- Whether Docker and Docker Compose are pre-installed, or must be provisioned.
- Domain name and whether an SSL certificate already exists, or if Let's Encrypt + nginx should be set up.
- Storage for Excel export files — local Docker volume (acceptable for single-node) or a blob storage bucket?
- Firewall/port rules: 80, 443 (nginx), 5432 (Postgres, internal only), 6379 (Redis, internal only).

**Implementation note:** Docker Compose setup is cloud-agnostic — no changes needed to the stack regardless of the underlying VM or cloud account.

---

#### Q9 — Proficiency Validation ✅ Answered · nemanjaninkovic-1

**Answer:** Proficiency is **self-reported by the employee** and enters a **"pending confirmation"** state until the Line Manager or Tech Lead confirms it.

**Behaviour:**

| State | Visible in search? | Profile badge |
| ----- | ------------------ | ------------- |
| `PENDING` | ✅ Yes — included in search with a "pending" indicator | ⏳ Awaiting confirmation |
| `CONFIRMED` | ✅ Yes — full weight in scoring | ✅ Confirmed |
| `REJECTED` | ❌ No — excluded from search | ❌ Rejected by manager |

**Rules:**

- Employee submits proficiency → status = `PENDING`, manager notified.
- Manager confirms → status = `CONFIRMED`; employee notified.
- Manager edits and saves a different value → new value is `CONFIRMED` immediately; old value replaced; employee notified.
- If manager never acts: skill stays `PENDING` indefinitely (no auto-expiry in Pillar 01).
- `LINE_MANAGER` or `TECH_LEAD` can confirm for their own subordinates / team members only.

**DB schema additions:**

```sql
ALTER TABLE employee_skills
  ADD COLUMN validation_status VARCHAR(10) NOT NULL DEFAULT 'PENDING'
    CHECK (validation_status IN ('PENDING', 'CONFIRMED', 'REJECTED')),
  ADD COLUMN validated_by UUID REFERENCES employees(id),
  ADD COLUMN validated_at TIMESTAMPTZ;
```

**Search impact:** Pending skills are indexed and searchable but the search result card shows a ⏳ badge. Embedding is rebuilt on every save regardless of status.

---

#### Q10 — UI Language ✅ Answered · ilija-radonjic

**Answer:** The platform UI will be available in four languages: **Serbian, English, Italian, Albanian**. The AI embedding model (`paraphrase-multilingual-MiniLM-L12-v2`) supports all four languages natively. Next.js `next-intl` will be used for i18n with locale-based routing.

---

#### Q11 — Dispute Flow for Manager-Edited Proficiency 🟡 Medium · nemanjaninkovic-1

**What we need to know:**

- If a manager lowers an employee's self-reported proficiency, can the employee formally dispute the change?
- If yes: who arbitrates — HR Coordinator, or the manager's own manager?
- What is the resolution timeline, and does the disputed value revert to self-reported during arbitration or stay at the manager-set value?
- Should disputes be tracked in the audit log with a dedicated event type?

**Why it matters:**
Currently the spec records all manager edits in the audit log and sends the employee a notification, but there is no dispute workflow. Adding a dispute system requires a new DB table (`skill_disputes`), new API endpoints, and additional UI. If disputes are out of scope for MVP, the notification must at minimum reference an offline process for employees to follow.

**Current assumption:** No in-platform dispute mechanism for MVP; employees are informed via notification and disputes are handled offline.

---

#### Q12 — SMTP Configuration for Email Notifications 🟡 Medium · nemanjaninkovic-1

**What we need to know:**

- Is there a company SMTP server, or should a transactional email service (SendGrid, SES, Mailgun) be used?
- SMTP host, port (587/465), TLS/SSL requirement, and auth credentials (username + password or API key).
- From address and display name for outgoing notifications (e.g., `noreply@company.com`).
- Are there email sending limits or domain restrictions (SPF/DKIM records to set up)?

**Why it matters:**
The Celery `send_notification` task currently supports both in-app and email channels. Without SMTP credentials the email channel falls back to in-app only, which may be acceptable for MVP but must be confirmed. If a transactional service is used, the task implementation changes (HTTP API vs. SMTP).

**Current assumption:** SMTP will be provided; in-app notifications are the fallback if it is not available before launch.

---

#### Q13 — Proficiency Format in Existing Excel Files � N/A

**Removed.** This question is superseded by Q4 — the actual Excel file will be provided by HR (see Q4). Format details will be determined from the real file.

---

#### Q14 — Search Filters ✅ Answered · engveselin

**Answer:** Search **must support structured filters** alongside free-text query. Filters are applied as SQL `WHERE` clauses before the HNSW vector similarity step (pre-filter).

**Supported filter dimensions:**

| Filter | Field | Source |
| ------ | ----- | ------ |
| Department | `employees.department` | Workday sync / Excel import |
| Position / Role type | `employees.title` | Workday sync / Excel import |
| Role category | e.g., "PM", "Dev", "Architect" | Derived from `title` or explicit field |

**Implementation notes:**

- Filters passed as optional params to `POST /api/search`: `department`, `title`.
- Pre-filtering reduces the HNSW candidate set before the `<=>` cosine operator.
- Redis cache key includes filter values: `search:{sha256(query + sorted(filters))}`.
- `employees.department` and `employees.title` must have B-tree indexes.
- Search UI exposes filter dropdowns populated from `GET /api/employees/filters` (distinct department + title values).

---

#### Q15 — Employee Profile Photos: Source and Storage 🟡 Medium · ilija-radonjic

**What we need to know:**

- Should profile photos be synced from Workday, pulled from **Azure AD** (via Microsoft Graph API), or uploaded manually by the employee?
- Accepted file formats and maximum file size (e.g., JPEG/PNG, max 2 MB)?
- Storage location: Azure Blob Storage, AWS S3, or local Docker volume (not recommended for production)?
- Should a fallback avatar (initials-based) be shown when no photo is available?

**Why it matters:**
Photo storage affects the backend (file upload endpoint, size/type validation, storage driver) and the frontend (avatar component, CDN URL vs. API-served bytes). If photos come from Workday or Azure AD, no upload endpoint is needed but the sync job grows in scope. Storage outside the DB volume is required for cloud deployments to avoid data loss on container restart.

**Current assumption:** No profile photo in Pillar 01 MVP; initials-based avatar placeholder shown.

---

#### Q16 — Accessibility and Compliance Requirements 🟡 Medium · ilija-radonjic

**What we need to know:**

- Is WCAG 2.1 Level AA the target, or is Level AAA required for any part of the UI?
- Are there GDPR obligations for employees in EU countries? If yes: is a Data Processing Agreement (DPA) in place, and is there a right-to-erasure workflow needed?
- Are there Serbian or Albanian labor law requirements for how employee competency data is stored, accessed, or deleted?
- Any screen reader testing targets (NVDA, JAWS, VoiceOver)?
- Is a cookie consent banner required (if any analytics are added later)?

**Why it matters:**
GDPR right-to-erasure conflicts with the soft-delete-only policy (`is_active = FALSE`). If hard deletion is required, the data model and audit log retention logic must change. Identifying compliance obligations early prevents costly retrofits after launch.

**Current assumption:** WCAG 2.1 AA only; GDPR compliance handled at the organizational level; no in-platform erasure workflow for Pillar 01.

---

#### Q17 — Workday Sync Failure SLA and Alerting 🟡 Medium · DusanEngIt

**What we need to know:**

- If the daily Workday sync at 02:00 fails after 3 retries, what should happen?
  - Option A: Fail silently — the platform continues with stale data until the next run.
  - Option B: Alert the engineering team via email or Slack webhook.
  - Option C: Alert HR Coordinator via in-app notification.
  - Option D: Block HR edits until sync is confirmed healthy.
- What is the acceptable staleness window for employee data (e.g., if sync fails for 3 consecutive nights, is that a P1 incident)?
- Is there a status page or health dashboard where sync state should be exposed?

**Why it matters:**
The current Celery task marks the job as `FAILED` and logs the error, but no external alert is sent. If the answer is Options B or C, a new alerting mechanism (webhook task or notification event type) must be added. Option D (blocking HR edits) requires a sync health check in the API middleware.

**Current assumption:** Silent fail with 3 retries; engineering monitors via Celery/Flower logs.

---

#### Q18 — Data Retention Policy for Audit Logs and Terminated Employees 🟡 Medium · DusanEngIt

**What we need to know:**

- How long should audit log records be retained (GDPR standard is 6–7 years for HR data)?
- For terminated employees (soft-deleted with `is_active = FALSE`): when, if ever, can their records be permanently deleted?
- Should there be an automated purge job (Celery beat task) or a manual HR action?
- Does the company have an existing data retention policy document that governs this?
- Do audit logs need to be exportable for compliance audits, or is DB access sufficient?

**Why it matters:**
Without a defined retention window, audit logs will grow indefinitely. A Celery beat purge task (e.g., delete entries older than 7 years) is straightforward to implement but must be scoped correctly to avoid inadvertently deleting records that are still within the retention window. Terminated employee records also contain personal data — holding them indefinitely may conflict with GDPR Article 17.

**Current assumption:** No automated purge in Pillar 01; data retained indefinitely pending policy confirmation.

---

## 18. Educated Guesses — Pending Decisions

> These are reasoned assumptions made in the absence of confirmed answers. They are used as the working basis for implementation. Each must be validated with stakeholders before UAT. If a guess turns out to be wrong, the affected section is noted for change impact.

---

### 18.1 Visibility Rules (Who Sees What)

| Actor | Own data | Subordinates / Team | Other employees (outside team) | Org-wide |
| ----- | -------- | ------------------- | ------------------------------ | -------- |
| `EMPLOYEE` | Full profile + audit history | — | Hierarchy only (name, dept, title, org chart) — no skill matrix | — |
| `LINE_MANAGER` | Full profile | Full profile + skill matrix | Hierarchy only — no skill matrix | — |
| `TECH_LEAD` | Full profile | Full profile + skill matrix | Hierarchy only — no skill matrix | — |
| `HR_COORDINATOR` | Full profile | Full profile + skill matrix | Full profile + skill matrix | Analytics dashboards |
| `GENERAL_MANAGEMENT` | Full profile | Full profile + skill matrix | Full profile + skill matrix | Analytics dashboards |

**Guesses applied:**

- Line Managers and Tech Leads **cannot** see skill matrices of employees outside their team. This is enforced server-side — the profile API strips skill data for non-subordinates (see `backend.instructions.md`).
- Employees can browse the org directory (name, department, title, org chart position) for all 1,400 employees, but **cannot** see anyone else's skill matrix.
- `GENERAL_MANAGEMENT` has **full read-only visibility** across all profiles and skill matrices, plus aggregate dashboards — but cannot edit anything.

**Change impact if wrong:** If Line Managers should see all skill matrices (e.g., for resource planning across teams), the `view_scope` logic in `app/auth/permissions.py` and the profile API response schema must be updated. No DB schema change needed.

---

### 18.2 Source-of-Truth Logic (SharePoint vs. Platform Edits)

The master skill data lives in an Excel file on SharePoint. The platform also allows employees and managers to edit skills directly. This creates a potential conflict on re-import.

**Educated guess — conflict resolution policy:**

| Scenario | Winner | Rationale |
| -------- | ------ | --------- |
| Employee edits skill in platform, HR later re-imports same employee row | **Platform wins** | Employee self-service should not be silently overwritten |
| Manager confirms a skill in platform, HR re-imports a different value | **Platform wins** | Confirmed values are authoritative |
| HR re-import contains a skill not yet in the platform | **Import wins** | New data, no conflict |
| HR explicitly triggers a "full override" import (admin flag) | **Import wins** | HR has override capability when needed |

**Implementation:** Each `employee_skill` row carries `last_modified_by_source ENUM('PLATFORM', 'IMPORT')` and `last_modified_at`. On import, a row is skipped if `last_modified_by_source = 'PLATFORM'` and `last_modified_at > import_file_created_at`, unless the import job has `force_override = TRUE`.

**Change impact if wrong:** If HR decides imports always win, remove the skip logic in `import_employees_task`. If they want per-field granularity, the schema needs a `source` column per skill field.

---

### 18.3 MVP Scope (Q7 — Educated Guess)

Q7 is officially open (DusanEngIt to confirm), but the 6-week timeline and team size imply a realistic MVP boundary. **Educated guess:**

| Feature | MVP (Pillar 01) | Deferred |
| ------- | --------------- | -------- |
| Azure AD SSO | ✅ | |
| Employee profile CRUD | ✅ | |
| Skill self-service (fill matrix) | ✅ | |
| Proficiency validation flow (PENDING/CONFIRMED) | ✅ | |
| Excel import from SharePoint | ✅ | |
| Semantic search + filters + pagination | ✅ | |
| In-app notifications | ✅ | |
| Review cycles — semi-annual + annual (HR-triggered) | ✅ | |
| Ad-hoc review cycles (Manager-triggered) | ✅ | |
| Custom skill matrices (Line Manager / Tech Lead) | ✅ | |
| Excel export | ✅ | |
| Custom roles (HR Coordinator) | ✅ | |
| Workday sync | ⚠️ Week 4 — may slip to Pillar 02 if Workday API credentials arrive late | |
| Email notifications (SMTP) | ⚠️ In-app only if SMTP not configured by Week 5 | |
| GENERAL_MANAGEMENT dashboards | ⚠️ Basic only for MVP | |
| Employee profile photos | ❌ | Pillar 02 |
| SharePoint direct integration (Graph API) | ❌ | Pillar 02 |
| Dispute workflow (in-platform) | ❌ | Pillar 02 |
| Mobile app | ❌ | Pillar 02+ |

**Change impact if wrong:** Removing Workday sync from MVP frees ~5 days of `nemanjaninkovic-1` time and removes the `workday_sync` Celery task, `workday_service.py`, and the `workday_id` sync logic. Schema stays the same (`workday_id` column is nullable).

---

### 18.4 Requirements vs. Implementation Separation

This spec intentionally mixes **what** the system must do (requirements) with **how** it should be built (implementation). The table below clarifies the boundary:

| Topic | Requirement (WHAT) | Implementation guess (HOW) |
| ----- | ------------------ | -------------------------- |
| Search returns ranked employees | ✅ Confirmed requirement | Hybrid score = 0.60/0.25/0.15; pgvector HNSW |
| Min match threshold (default 70%) | ✅ Confirmed requirement | `hybrid_score >= settings.search_min_score`; HR-configurable via `PATCH /api/settings/search` |
| Page size (default 10) | ✅ Confirmed requirement | `settings.search_page_size`; HR-configurable via `PATCH /api/settings/search` |
| Proficiency is self-reported + confirmed | ✅ Confirmed requirement | `validation_status ENUM` on `employee_skills` |
| Filters by department / title | ✅ Confirmed requirement | SQL `WHERE` pre-filter before HNSW |
| Source of truth = SharePoint Excel | ✅ Confirmed requirement | Platform-wins conflict policy — **guess** |
| Workday sync at 02:00 | Requirement — Workday type TBC | Celery beat + `workday_service.py` — **guess** |
| Custom roles | ✅ Confirmed requirement | JSONB permissions + union logic — **guess** |
| Email notifications | Requirement — SMTP TBC | Celery `send_notification` task — **guess** |
| Azure AD SSO | ✅ Confirmed requirement | MSAL + JWKS validation — **guess on library** |

**Rule:** If a stakeholder changes a ✅ requirement, re-estimate the change impact. If they change a **guess**, it is likely a low-cost swap in one service or task.

---

## 19. Milestones

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
