---
description: "Use when writing Next.js components, pages, API client code, or styles in apps/frontend/. Covers App Router conventions, typed API client, auth, CSS variables, and proficiency display rules."
applyTo: "apps/frontend/**"
---

# Frontend Conventions

## App Router Layout

```text
src/
├── app/                          # Next.js App Router — file-based routing
│   ├── layout.tsx
│   ├── page.tsx                  # dashboard
│   ├── search/page.tsx
│   ├── profile/[id]/page.tsx
│   ├── profile/edit/page.tsx
│   ├── import/page.tsx
│   ├── export/page.tsx
│   └── admin/{taxonomy,reviews}/page.tsx
├── components/
│   ├── ui/                       # Button, Badge, SkillCard, ProfileCard, SearchBar
│   ├── import/                   # FileUpload, ColumnMapper, ValidationTable, ImportProgress
│   ├── export/                   # ExportForm, ExportProgress, ExportDownload
│   └── layout/                   # Navbar, Sidebar
├── styles/globals.css            # CSS variables — brand colors defined here
└── lib/
    ├── api.ts                    # typed API client
    ├── auth.ts                   # NextAuth.js
    └── types.ts
```

## API Calls

- **Always** use `src/lib/api.ts` — never call `fetch` directly in components
- All responses are typed via `src/lib/types.ts`

## Styling

- Brand colors live exclusively in `src/styles/globals.css` as CSS variables
- **Never** hardcode hex values in components — always reference `var(--color-*)`

```css
/* Available variables */
--color-primary, --color-primary-dark, --color-secondary
--color-background, --color-surface
--color-text-primary, --color-text-secondary
--color-success, --color-warning, --color-error
```

## Proficiency Display

Proficiency is always an integer 1–5. Display label alongside number:

| Value | Label        |
| ----- | ------------ |
| 1     | Beginner     |
| 2     | Elementary   |
| 3     | Intermediate |
| 4     | Advanced     |
| 5     | Expert       |

Use `<Badge>` component for proficiency levels — do not render raw numbers without labels.

## Auth

- Session managed via NextAuth.js (`src/lib/auth.ts`)
- Role available on session object: `EMPLOYEE | LINE_MANAGER | HR_COORDINATOR | GENERAL_MANAGEMENT`
- Gate UI elements by role — do not rely solely on backend for access control visibility

## Import Wizard (Excel)

Status polling: `GET /api/import/{job_id}/status` — use `ImportProgress` component which handles polling internally. Do not implement custom polling logic.

## Export (Excel)

- Trigger: `POST /api/export` — scope is determined server-side from JWT role; do not pass role in the request body
- Status polling: `GET /api/export/{job_id}/status` — use `ExportProgress` component
- Download: `GET /api/export/{job_id}/download` — link shown only when status is `COMPLETED`
- Show the Export button only to roles that have export scope (`EMPLOYEE`, `LINE_MANAGER`, `HR_COORDINATOR`); hide for `GENERAL_MANAGEMENT`
