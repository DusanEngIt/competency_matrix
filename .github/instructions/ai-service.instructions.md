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

Never change these weights without benchmarking against the full 1,400-profile dataset.

## pgvector Query Pattern

```sql
-- ANN search with HNSW index (~5ms at 1,400 rows)
SELECT id, 1 - (profile_embedding <=> $1::vector) AS cosine_score
FROM employees
WHERE is_active = TRUE
ORDER BY profile_embedding <=> $1::vector
LIMIT 50;
```

## Profile Vector

- `profile_embedding vector(384)` on `employees` table = weighted mean of all skill embeddings
- Rebuilt asynchronously via Celery task `embed_profile.py` on every profile save
- Do **not** rebuild synchronously in the request path — always enqueue via Celery

## Libraries in Use

`fastapi` · `uvicorn` · `sentence-transformers` · `asyncpg` · `sqlalchemy[asyncio]` · `pgvector` · `rank-bm25`
