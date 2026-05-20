---
description: "Use when writing embedding, vector search, or hybrid re-ranking code in apps/ai-service/. Covers model usage, vector dimensions, pgvector queries, hybrid scoring, and Redis caching."
applyTo: "apps/ai-service/**"
---

# AI Service Conventions

## Service Endpoints

| Method | Path      | Purpose                                                |
| ------ | --------- | ------------------------------------------------------ |
| `POST` | `/embed`  | Generate 384-dim embedding for text                    |
| `POST` | `/search` | Semantic + hybrid search, returns ranked employee list |

## Model

- **Model:** `paraphrase-multilingual-MiniLM-L12-v2`
- **Vector dimension:** `384` — fixed; do not change without rebuilding all stored embeddings
- **Languages:** Serbian + English (mixed queries work natively)
- **RAM:** ~420MB; runs on CPU; ~10ms per embedding
- Model weights cached to `/app/models/` Docker volume — loaded once on startup

## Hybrid Search Score

```python
score = 0.7 * vector_cosine_similarity + 0.3 * ts_rank
```

- **Minimum threshold: 0.70** — discard any result with `hybrid_score < 0.70` before returning.
- **Page size: 10** — return at most 10 results per page.
- Never change the score weights without benchmarking against the full 1,400-profile dataset.

## pgvector Query Pattern

Search supports optional pre-filters on `department` and `title`. Apply them as `WHERE` clauses **before** the `<=>` vector operator so the HNSW index scans a smaller candidate set:

```sql
-- ANN search with optional pre-filters; over-fetch then filter by score
SELECT id, 1 - (profile_embedding <=> $1::vector) AS cosine_score
FROM employees
WHERE is_active = TRUE
  AND ($2::text IS NULL OR department = $2)
  AND ($3::text IS NULL OR title ILIKE '%' || $3 || '%')
ORDER BY profile_embedding <=> $1::vector
LIMIT 100;  -- over-fetch; hybrid re-ranking + 0.70 threshold applied in Python
```

- Over-fetch 100 candidates from pgvector, apply hybrid re-ranking in Python, then filter `score >= 0.70`.
- Paginate the filtered list at **10 per page**; return `{results, page, total_pages, total_count}`.
- `department` and `title` accept `None` to skip the filter (no-op).
- Redis cache key: `search:{sha256(query + json.dumps(sorted_filters))}:page:{n}`, TTL 5 min.
- `GET /api/employees/filters` returns distinct `department` and `title` values for search UI dropdowns.

## Profile Vector

- `profile_embedding vector(384)` on `employees` table = weighted mean of all skill embeddings
- Rebuilt asynchronously via Celery task `embed_profile.py` on every profile save
- Do **not** rebuild synchronously in the request path — always enqueue via Celery

## Libraries in Use

`fastapi` · `uvicorn` · `sentence-transformers` · `asyncpg` · `sqlalchemy[asyncio]` · `pgvector` · `rank-bm25`
