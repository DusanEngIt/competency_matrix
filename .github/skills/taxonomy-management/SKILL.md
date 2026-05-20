---
name: taxonomy-management
description: "Manage the skill taxonomy: add new skills, deprecate old ones, review and approve pending skill proposals from imports or employee free-text input, check for duplicates, and invalidate taxonomy cache. Use when HR needs to update taxonomy or clear a backlog of unreviewed skill proposals."
argument-hint: "Describe the task: 'add skill', 'deprecate skill', 'review proposals', or 'find duplicates'"
---

# Taxonomy Management

Guided workflow for HR Coordinators to safely add, deprecate, and review taxonomy skills.

## Adding a New Skill

```sql
-- 1. Check for duplicates first
SELECT id, name, category, is_active
FROM skills
WHERE similarity(name, '<new_skill_name>') > 0.4
   OR lower(name) LIKE lower('%<keyword>%');
```

If no duplicate:

```bash
# Via API (preferred — auto-generates embedding)
curl -X POST http://localhost:8000/api/skills \
  -H "Authorization: Bearer <HR_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Kubernetes",
    "category": "HARD",
    "description": "Container orchestration platform"
  }'
```

After adding, invalidate taxonomy cache:

```bash
docker compose exec redis redis-cli keys "taxonomy:*" | xargs docker compose exec redis redis-cli del
```

## Deprecating a Skill

Soft-delete only — never hard-delete (employees may reference it):

```sql
-- 1. Check how many employees use this skill
SELECT COUNT(*) FROM employee_skills
WHERE skill_id = '<skill_id>' AND proficiency_level IS NOT NULL;

-- 2. Soft-delete
UPDATE skills SET is_active = FALSE WHERE id = '<skill_id>';
```

Then bust cache:

```bash
docker compose exec redis redis-cli keys "taxonomy:*" | xargs docker compose exec redis redis-cli del
docker compose exec redis redis-cli keys "search:*" | xargs docker compose exec redis redis-cli del
```

## Reviewing Pending Skill Proposals

Skills proposed from import or employee free-text have `approved_by IS NULL` and `is_active = FALSE`:

```sql
SELECT id, name, category, created_at, created_by,
       (SELECT email FROM employees WHERE id = s.created_by) AS proposed_by
FROM skills s
WHERE approved_by IS NULL AND is_active = FALSE
ORDER BY created_at ASC;
```

To approve:

```sql
UPDATE skills
SET is_active = TRUE, approved_by = '<hr_employee_id>'
WHERE id = '<skill_id>';
```

To reject (leave inactive, optionally map to existing skill):

```sql
-- Map rejected proposal to existing taxonomy entry in employee_skills
UPDATE employee_skills
SET skill_id = '<existing_skill_id>'
WHERE skill_id = '<proposed_skill_id>';

-- Then delete the proposal (only safe after reassigning all references)
DELETE FROM skills WHERE id = '<proposed_skill_id>' AND approved_by IS NULL;
```

## Finding Duplicates

```sql
SELECT name, category, COUNT(*) AS count
FROM skills
WHERE is_active = TRUE
GROUP BY name, category
HAVING COUNT(*) > 1;

-- Fuzzy duplicates (pg_trgm)
SELECT a.id, a.name, b.id, b.name, similarity(a.name, b.name) AS sim
FROM skills a, skills b
WHERE a.id < b.id
  AND a.category = b.category
  AND similarity(a.name, b.name) > 0.7
  AND a.is_active = TRUE AND b.is_active = TRUE
ORDER BY sim DESC;
```

## After Any Taxonomy Change

Always bust both caches:

```bash
docker compose exec redis redis-cli keys "taxonomy:*" | xargs docker compose exec redis redis-cli del
docker compose exec redis redis-cli keys "search:*" | xargs docker compose exec redis redis-cli del
```
