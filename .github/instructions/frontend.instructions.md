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
/* ENG Brand — extracted from eng.it production CSS */
--color-primary: #c51b88; /* ENG magenta — buttons, links, active */
--color-primary-dark: #8c1e74; /* hover / pressed */
--color-primary-light: #f4d1e8; /* badges, subtle tints */
--color-dark-navy: #323551; /* headers, sidebar */
--color-near-black: #161721; /* dark surfaces */
--color-background: #fcfcfc; /* page background */
--color-surface: #ffffff; /* cards, modals */
--color-text-primary: #090909; /* body text */
--color-text-secondary: #9d9d9d; /* secondary text, borders */
--color-success: #198754;
--color-warning: #ffc107;
--color-error: #da1e28;
--color-info: #323551;
```

## Proficiency Display

Proficiency is always an integer 1–5. Display label alongside number:

| Value | Label        |
| ----- | ------------ |
| 1     | Beginner     |
| 2     | Intermediate |
| 3     | Experienced  |
| 4     | Advanced     |
| 5     | Master       |

Use `<Badge>` component for proficiency levels — do not render raw numbers without labels.

## Auth

- Session managed via NextAuth.js (`src/lib/auth.ts`)
- Built-in role available on session object: `EMPLOYEE | LINE_MANAGER | TECH_LEAD | HR_COORDINATOR | GENERAL_MANAGEMENT`
- Session also exposes `customPermissions` object (from `/api/auth/me`) for any custom role permissions assigned to the user
- Gate UI elements by role — do not rely solely on backend for access control visibility
- Custom role management UI (create/edit/delete roles, assign to employees) is visible **only to `HR_COORDINATOR`**

## Import Wizard (Excel)

Status polling: `GET /api/import/{job_id}/status` — use `ImportProgress` component which handles polling internally. Do not implement custom polling logic.

## Export (Excel)

- Trigger: `POST /api/export` — scope is determined server-side from JWT role; do not pass role in the request body
- Status polling: `GET /api/export/{job_id}/status` — use `ExportProgress` component
- Download: `GET /api/export/{job_id}/download` — link shown only when status is `COMPLETED`
- Show the Export button only to roles that have export scope (`EMPLOYEE`, `LINE_MANAGER`, `HR_COORDINATOR`); hide for `GENERAL_MANAGEMENT`
