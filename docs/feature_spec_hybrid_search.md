# Feature Spec: Hybrid Search Engine

**Feature:** Поисковый движок для хранилища знаний пользователя  
**Priority:** P0 (критический путь — без этого не работает inference, RAG, confidence gating)  
**Owner:** Server Team (MVP), Mobile Team (V2 — оффлайн)  
**Status:** Draft  

---

## 1. Проблема

Jippy хранит ВСЕ данные пользователя как Q&A пары. При каждом вопросе нужно быстро найти релевантные знания из хранилища. Это используется для:
- **Confidence gating:** "знакомая тема или нет?" → решает инджектить hint или нет
- **RAG retrieval:** достать новые знания (после level-up) для дополнения модели
- **Import dedup:** проверить что Q&A пара ещё не существует

Два типа поиска решают разные задачи:
- Vector (semantic): "диабет лечение" найдёт "сахарный диабет терапия" — смысл совпадает
- Keyword (BM25): "метформин 500мг" найдёт точное вхождение — embedding может пропустить

Только вместе дают 96.6% retrieval accuracy (подтверждено MemPalace на LongMemEval бенчмарке).

---

## 2. Решение

Hybrid Search: два параллельных поиска + объединённый ranking.

---

## 3. Как работает

### Схема

```
ЗАПРОС: "какая дозировка метформина при диабете 2 типа?"
                        │
          ┌─────────────┴─────────────┐
          ▼                           ▼
   BM25 Keyword Search         Vector Semantic Search
          │                           │
   Ищет точные слова:          Ищет по смыслу:
   "метформин"                 embedding(запрос) vs
   "дозировка"                 embedding(все Q&A)
   "диабете"                          │
          │                           │
          ▼                           ▼
   Результаты:                 Результаты:
   1. "метформин 500мг         1. "лечение сахарного
      при СД2" (score 8.2)        диабета" (sim 0.89)
   2. "дозировка инсулина"     2. "препараты при
      (score 3.1)                 диабете 2 типа" (0.85)
   3. "метформин побочные"     3. "метформин назначение
      (score 2.8)                 эндокринолог" (0.82)
          │                           │
          └─────────────┬─────────────┘
                        ▼
              COMBINED RANKING
              ──────────────
              Для каждого документа:
              final_score = α × norm(bm25) + (1-α) × vector_sim

              α = 0.4 (по умолчанию, настраиваемый)

              Топ-K результатов → возврат
```

### Формулы

**BM25 (Okapi BM25):**
```
score(q, d) = Σ IDF(qi) × (tf(qi,d) × (k1+1)) / (tf(qi,d) + k1 × (1 - b + b × |d|/avgdl))

где:
  qi = термин из запроса
  tf = term frequency в документе
  IDF = log((N - df + 0.5) / (df + 0.5) + 1)
  k1 = 1.5 (term frequency saturation)
  b = 0.75 (length normalization)
  N = кол-во документов
  df = document frequency термина
  avgdl = средняя длина документа
```

**Vector Similarity:**
```
sim(q, d) = cosine(embedding(q), embedding(d))
          = dot(eq, ed) / (|eq| × |ed|)
```

**Combined Score:**
```
final(q, d) = α × normalize(bm25(q,d)) + (1-α) × vector_sim(q,d)

normalize(x) = (x - min) / (max - min)  — min-max по результатам этого запроса
α = 0.4 по умолчанию
```

---

## 4. Два контекста использования

### 4.1 Confidence Gating (каждый запрос)

**Задача:** Быстро определить — знакомая тема или нет.

```
POST /api/jippy/search
Body: { 
  query: "какая дозировка метформина?",
  limit: 5,
  threshold: 0.6,
  search_scope: "full_storage"   ← ищем по ВСЕМУ хранилищу
}

Response: {
  results: [
    { id: "...", question: "...", answer: "...", score: 0.91 },
    { id: "...", question: "...", answer: "...", score: 0.84 },
    ...
  ],
  is_familiar: true,            ← score[0] > threshold
  top_score: 0.91
}
```

**Логика:**
- `top_score > 0.6` → знакомая тема → Jippy генерирует domain hint
- `top_score < 0.6` → незнакомая тема → backbone отвечает сам
- Порог 0.6 — стартовое значение, калибруем на реальных данных

### 4.2 RAG Retrieval (дополнение модели)

**Задача:** Достать конкретные знания для передачи модели.

```
POST /api/jippy/search
Body: { 
  query: "какая дозировка метформина?",
  limit: 3,
  threshold: 0.5,
  search_scope: "rag_buffer"    ← ищем ТОЛЬКО в новых (после level-up)
}

Response: {
  results: [
    { id: "...", question: "...", answer: "...", score: 0.87 }
  ],
  is_familiar: true,
  top_score: 0.87
}
```

**Логика:**
- `search_scope: "full_storage"` — для confidence gating (знакомая тема?)
- `search_scope: "rag_buffer"` — для RAG retrieval (новые знания для hint)
- Оба используют один и тот же hybrid search, разная область поиска

---

## 5. Deployment: Server vs Mobile

### MVP: Server Only

Вся логика на сервере. Телефон отправляет запрос → сервер ищет → возвращает результаты.

```
Телефон                         Сервер
  │                               │
  │  POST /api/jippy/search       │
  │  { query, scope }             │
  │──────────────────────────────►│
  │                               │  PostgreSQL FTS (BM25)
  │                               │  + pgvector (embeddings)
  │                               │  + combined ranking
  │                               │
  │  { results, is_familiar }     │
  │◄──────────────────────────────│
  │                               │
```

**Стек:**
- PostgreSQL FTS (full-text search) — BM25
- pgvector extension — vector similarity search
- Один SQL запрос с CTE: BM25 scores + vector scores + combined ranking
- Latency: <100ms на 100K документов

**Embedding model (сервер):**
- `all-MiniLM-L6-v2` (22M params, 384-dim embeddings)
- Или `multilingual-e5-small` (118M params, 384-dim) — лучше для русского

### V2: Server + Mobile Offline

Телефон имеет локальную копию для оффлайн работы.

```
Телефон (оффлайн)               Сервер (онлайн)
  │                               │
  │  SQLite FTS5 (BM25)           │  PostgreSQL FTS
  │  + local embeddings           │  + pgvector  
  │  + MiniLM-L6 (22MB on-device) │  + full embedding model
  │                               │
  │  Scope: RAG buffer only       │  Scope: full storage
  │  (для быстрого ответа)        │  (для confidence gating)
  │                               │
```

**Телефонный стек:**
- SQLite FTS5 — встроен в iOS/Android, zero dependencies
- `all-MiniLM-L6-v2` — 22MB, ONNX Runtime inference
- Scope: только RAG buffer (компактный, быстрый)
- Latency: <50ms

**Когда оффлайн:**
- Confidence gating по RAG buffer (не по full storage)
- Менее точный, но работает без интернета
- Backbone недоступен → Jippy отвечает сам (без server backbone)

---

## 6. Индексация

### При добавлении Q&A пары (каждый запрос / импорт)

```
Новая Q&A пара приходит
         │
         ▼
1. Генерация embedding:
   embedding = encode(question + " " + answer)
   Модель: all-MiniLM-L6-v2 (или multilingual-e5-small)
   Размер: 384 float values
         │
         ▼
2. Индексация для BM25:
   PostgreSQL: INSERT с tsvector(question || answer)
   SQLite: INSERT в FTS5 virtual table
         │
         ▼
3. Сохранение:
   Q&A pair + embedding → permanent storage
   Q&A pair + embedding → RAG buffer (если после level-up)
```

### При level-up

```
Level-up завершён
         │
         ▼
1. RAG buffer очищается (index reset)
2. Full storage index остаётся без изменений
3. Новые Q&A будут индексироваться в свежий RAG buffer
```

---

## 7. API

```
POST /api/jippy/search
  Headers: Authorization: Bearer <jwt>
  Body: {
    "query": "текст запроса",
    "limit": 5,                           // макс результатов (default 5)
    "threshold": 0.6,                      // мин score (default 0.6)
    "search_scope": "full_storage" | "rag_buffer",  // где искать
    "alpha": 0.4                           // вес BM25 vs vector (default 0.4)
  }
  
  Response: {
    "results": [
      {
        "id": "sha256-hash",
        "question": "текст вопроса",
        "answer": "текст ответа",
        "score": 0.91,                     // combined score
        "bm25_score": 0.85,               // normalized BM25
        "vector_score": 0.94,             // cosine similarity
        "source": "chat | import_chatgpt",
        "timestamp": "2025-06-15T14:30:00Z"
      }
    ],
    "is_familiar": true,                   // top_score > threshold
    "top_score": 0.91,
    "total_searched": 12847,               // сколько документов просмотрено
    "latency_ms": 42
  }
```

---

## 8. Пример SQL (PostgreSQL MVP)

```sql
-- Combined hybrid search in one query
WITH bm25 AS (
  SELECT id, question, answer, source, timestamp,
         ts_rank_cd(search_vector, plainto_tsquery('russian', $1)) AS bm25_score
  FROM jippy_knowledge
  WHERE user_id = $2
    AND ($3 = 'full_storage' OR created_at > $4)  -- scope filter
    AND search_vector @@ plainto_tsquery('russian', $1)
  ORDER BY bm25_score DESC
  LIMIT 50
),
vector AS (
  SELECT id, question, answer, source, timestamp,
         1 - (embedding <=> $5::vector) AS vector_score  -- cosine similarity
  FROM jippy_knowledge
  WHERE user_id = $2
    AND ($3 = 'full_storage' OR created_at > $4)
  ORDER BY embedding <=> $5::vector
  LIMIT 50
),
combined AS (
  SELECT 
    COALESCE(b.id, v.id) AS id,
    COALESCE(b.question, v.question) AS question,
    COALESCE(b.answer, v.answer) AS answer,
    COALESCE(b.source, v.source) AS source,
    COALESCE(b.timestamp, v.timestamp) AS timestamp,
    COALESCE(b.bm25_score, 0) AS bm25_score,
    COALESCE(v.vector_score, 0) AS vector_score,
    $6 * COALESCE(b.bm25_score, 0) / NULLIF(MAX(b.bm25_score) OVER (), 0)
      + (1 - $6) * COALESCE(v.vector_score, 0) AS final_score
  FROM bm25 b
  FULL OUTER JOIN vector v ON b.id = v.id
)
SELECT * FROM combined
WHERE final_score > $7  -- threshold
ORDER BY final_score DESC
LIMIT $8;  -- limit
```

Параметры:
- $1: query text
- $2: user_id
- $3: search_scope ('full_storage' | 'rag_buffer')
- $4: last_levelup_timestamp (для scope фильтра)
- $5: query embedding vector
- $6: alpha (BM25 weight, default 0.4)
- $7: threshold (default 0.6)
- $8: limit (default 5)

---

## 9. Требования к БД

### PostgreSQL (сервер)

```sql
-- Расширения
CREATE EXTENSION IF NOT EXISTS vector;        -- pgvector для embeddings
CREATE EXTENSION IF NOT EXISTS pg_trgm;       -- триграммы для fuzzy match

-- Таблица знаний
CREATE TABLE jippy_knowledge (
  id TEXT PRIMARY KEY,                        -- sha256 hash
  user_id UUID NOT NULL,
  question TEXT NOT NULL,
  answer TEXT NOT NULL,
  raw_question TEXT NOT NULL,                 -- verbatim оригинал
  raw_answer TEXT NOT NULL,                   -- verbatim оригинал
  source TEXT NOT NULL,                       -- chat, import_chatgpt, etc
  timestamp TIMESTAMPTZ NOT NULL,
  topic_keywords TEXT[],
  embedding vector(384),                      -- MiniLM-L6 dimensions
  search_vector tsvector                      -- для FTS
    GENERATED ALWAYS AS (
      to_tsvector('russian', question || ' ' || answer)
    ) STORED,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Индексы
CREATE INDEX idx_knowledge_user ON jippy_knowledge(user_id);
CREATE INDEX idx_knowledge_fts ON jippy_knowledge USING gin(search_vector);
CREATE INDEX idx_knowledge_vector ON jippy_knowledge USING ivfflat(embedding vector_cosine_ops);
CREATE INDEX idx_knowledge_created ON jippy_knowledge(user_id, created_at);  -- для scope filter
```

### SQLite + FTS5 (телефон, V2)

```sql
-- Основная таблица
CREATE TABLE knowledge (
  id TEXT PRIMARY KEY,
  question TEXT,
  answer TEXT,
  source TEXT,
  timestamp TEXT,
  embedding BLOB  -- 384 × 4 bytes = 1536 bytes per row
);

-- FTS5 виртуальная таблица для BM25
CREATE VIRTUAL TABLE knowledge_fts USING fts5(
  question, answer, content=knowledge, content_rowid=rowid
);
```

---

## 10. Embedding Model

| Вариант | Размер | Dims | Язык | Где |
|---------|--------|------|------|-----|
| `all-MiniLM-L6-v2` | 22MB | 384 | EN (OK для RU) | Телефон + Сервер |
| `multilingual-e5-small` | 118MB | 384 | 100+ языков | Сервер (тяжеловат для телефона) |
| `paraphrase-multilingual-MiniLM-L12-v2` | 118MB | 384 | 50+ языков | Сервер |

**MVP:** `all-MiniLM-L6-v2` на сервере и телефоне (одна модель, одинаковые embeddings, простая синхронизация).

**V2:** `multilingual-e5-small` на сервере (лучше для русского), `MiniLM-L6` на телефоне (22MB компактнее). Нужна re-indexация при смене модели.

**Важно:** embedding модель на сервере и телефоне должна быть ОДИНАКОВОЙ (или хотя бы одной размерности), иначе поиск по synced данным не будет работать.

---

## 11. Метрики

| Метрика | Target |
|---------|--------|
| Retrieval accuracy (R@5) | >90% |
| Latency серверный поиск (100K docs) | <100ms |
| Latency телефонный поиск (10K docs) | <50ms |
| Confidence gating precision | >85% (правильно определяет знакомую тему) |
| Index size per 10K Q&A pairs | <50MB (embeddings + FTS) |

## 12. Roadmap

| Этап | Scope | Владелец |
|------|-------|----------|
| **MVP** | Server-side hybrid search (PostgreSQL + pgvector) | Server Team |
| **V1** | Confidence gating integration + threshold calibration | Server Team |
| **V2** | Mobile offline search (SQLite FTS5 + on-device MiniLM) | Mobile Team |
| **V3** | Embedding model upgrade (multilingual-e5) + re-indexation | Server Team |
