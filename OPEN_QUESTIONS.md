# Open Questions — Workforce Platform

> **For stakeholders.**
> These items are blocking or significantly affecting design decisions.
> Each question needs a confirmed answer before the related component can be built.
> Please provide answers to **🔴 High priority items first** — they block Week 1–2 work.

---

## 🔴 High Priority — Blockers

### 1. Workday Integration

**Question:** What type of Workday API is available — REST, RAAS, or SOAP?

**Why it matters:** Determines the entire Workday sync implementation. REST and RAAS require different auth flows and data extraction methods.

**Also needed:**
- Workday tenant URL
- OAuth2 client ID + secret (or equivalent credentials)
- Confirmation that a daily sync at 02:00 is acceptable

**Owner:** nemanjaninkovic-1

---

### 2. SSO / Authentication Provider

**Question:** Which SSO provider is in use — Azure AD, Okta, LDAP, or none?

**Why it matters:** If SSO is required, the auth service must integrate with the provider before any other feature can be tested by employees. If no SSO, username/password login is used.

**Owner:** nemanjaninkovic-1

---

### 3. Brand Colors

**Question:** What are the company's brand hex color values?

**Why it matters:** All UI components use CSS variables. Without the actual colors, the frontend ships with placeholders and requires a full visual pass before UAT.

**Needed:**
- Primary color
- Secondary / accent color
- Any logo or brand guidelines document

**Owner:** ilija-radonjic

---

### 4. Sample Excel Export from HR

**Question:** Can HR provide a sample (anonymized) Excel file of the current employee skill data?

**Why it matters:** The column mapping wizard and import parser are built around the actual column names in use. Without a sample, the import wizard will require rework after the first real import.

**Owner:** nemanjaninkovic-1

---

## 🟡 Medium Priority — Needed Before UAT

### 5. Cloud Infrastructure

**Question:** Is there an existing VM or cloud subscription to deploy on, or should the team provision a new one?

**Recommendation:** Azure B4ms (4 vCPU, 16GB RAM, ~$80/mo) — fits all containers comfortably.

**Owner:** nemanjaninkovic-1

---

### 6. Proficiency Validation — Self-Reported or Manager-Validated?

**Question:** Is a skill proficiency level set by the employee alone, or must the line manager confirm it?

**Current assumption:** Employee sets their own level; manager can override (with notification). Annual review cycles require manager sign-off.

**Why it matters:** If all edits require manager approval, a pending/approval workflow must be built. This significantly increases scope.

**Owner:** nemanjaninkovic-1

---

### 7. UI Language

**Question:** Should the platform UI be in Serbian, English, or both (switchable)?

**Why it matters:** Affects all frontend labels, error messages, and email notification templates. Dual-language support requires i18n setup.

**Current assumption:** Serbian primary, English secondary (skill names stored in English for search consistency).

**Owner:** ilija-radonjic

---

### 8. Dispute Flow

**Question:** When an employee disagrees with a skill edit made by their manager, what happens?

**Current assumption:** Employee sees a "Dispute this change" button in the notification → opens an HR support ticket (external system). No in-platform dispute tracking in Pillar 01.

**If a different flow is needed (e.g., in-platform dispute queue), this must be defined before the notification system is built.**

**Owner:** nemanjaninkovic-1

---

### 9. SMTP / Email Server

**Question:** Is there an SMTP server available for sending email notifications?

**Needed:**
- SMTP host + port
- Username / password or API key
- Sender address (e.g., `noreply@company.com`)

**Impact if missing:** Email notifications are disabled; only in-app alerts work until SMTP is configured.

**Owner:** nemanjaninkovic-1

---

### 10. Current Excel Format — Proficiency as Numbers or Text?

**Question:** In the existing Excel files, is proficiency stored as a number (1–5) or as text (e.g., Junior, Senior, Intermediate)?

**Why it matters:** The import parser auto-converts known text labels to numbers. If the existing files use custom text labels not in the standard map, they must be added before the first import run.

**Owner:** nemanjaninkovic-1

---

## How to Respond

Please reply to each question in this document or send answers to the engineering team directly.
For urgent items (🔴), target a response by **end of Week 1** to avoid blocking development.
