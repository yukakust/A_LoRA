# Implementation Plan — Jippy Backend MVP

**Статус:** активный (2026-04-29)
**Source of truth для:** порядок работы по бэкенду Jippy v1
**Связь со спекой:** реализует §17 Memory Architecture из `jippy_architecture_final.md`

---

## ✅ Done so far (2026-04-29 / sprint S1+S2+S6 + UI)

Краткая сводка реализованного — детали ниже по фазам.

**Sprint S1 — Layer 2c rebuild с NER:** ✅
- `extract_concepts.py` v2: spaCy NER (`ru_core_news_sm` + `en_core_web_sm`, lang-aware routing)
- NER labels: PERSON / LOC / GPE / ORG / PRODUCT / EVENT / WORK_OF_ART; PoS fallback NOUN/PROPN
- Lemma-canonical concepts + surface-form aliases; TF-IDF importance score
- Background rebuild job: `POST /concepts/rebuild` → 202 + job_id, `GET /concepts/rebuild/{id}` for progress + ETA
- Schema delta (`003_closeness_concepts.sql`): concepts.ner_type, importance, user_hidden, pinned_to_core
- User curation endpoints: hide / unhide / pin / merge / add_user_edge
- Graceful fallback to legacy frequency mode if spaCy missing

**Sprint S2 — Closeness multi-signal scoring:** ✅
- `extract_people.py`: 8-signal weighted closeness score (reply latency, self-initiated, emotional, bidirectionality, time-of-day, length, diversity, persistence)
- Lexical RU+EN emotional markers (~150 patterns); personal hours = 21:00–02:00 + weekends
- Tier mapping: close ≥0.75 / warm 0.50 / acquaint 0.25 / transactional <0.25
- Schema delta: people.closeness_score / closeness_tier / user_override / override_at
- `POST /people/:id/tier` (override) + `DELETE /people/:id/tier` (clear); auto-recompute respects user_override
- `GET /people?tier=close` filter; `GET /person/:id?with_signals=true` returns 8-signal breakdown

**Sprint S6 — Harness v2 parallel fetch:** ✅
- Replaced regex intent classifier + 5 per-intent assemblers with **single rich-context** flow
- ThreadPoolExecutor parallel fetch of Layer 1 / 2a / 2b / 2c / 2e (~100ms p50)
- Layer 0 split: 0a parametric (default, free) vs 0b Sonar (only on temporal markers — «сегодня/новости/курс/погода/now/today/latest»)
- NER-based names extraction from query (spaCy PERSON if loaded; else capitalize-word fallback)
- Lemma-based concept lookup in query via `concept_aliases.alias IN (...)` SQL
- Adaptive context window: char-budgeted (30K chars ≈ 10K tokens) — verbatim if fits, else `summary(older) + last 2 turns verbatim` via Cerebras backbone
- Single `_build_system_v2(profile, lang)` builder; explicit section headers in prompt («=== ФАКТЫ / ЛЮДИ / КОНЦЕПТЫ / ВОСПОМИНАНИЯ ===»)
- Removed: `_INTENT_PATTERNS`, `_classify_intent_heuristic`, `_assemble_*` (×5), `_build_*_system` (×5), `SYSTEM_TEMPLATES`, hard-rules check (returns in Phase Б with UI)
- End-to-end verified: route_latency ~1s (was ~2.2s), all 5 layers fetched in single pass

**Sprint S3+S4+S5 — UI tabs in `jippy_chat.html`:** ✅
- Tab nav: Chat / Кто я / Друзья / Карта идей
- **Profile tab «Кто я»**: 3 sections (ФАКТЫ / ВЕРОЯТНО / ДОГАДКИ) с tier-badges + confidence; inline edit/confirm/reject; manual add via select+input; Re-extract button
- **People tab «Друзья»**: 4-column kanban (close/warm/acquaint/transactional) с HTML5 drag-and-drop = manual tier override; per-card score + interactions + override pill; click → modal с 8-signal breakdown
- **Map tab «Карта идей»**: cytoscape.js force-directed graph; node = концепт, размер = importance, цвет = NER-type (PERSON purple / LOC green / ORG blue / PRODUCT amber / EVENT pink / WORK red / NOUN slate); click → expand depth-1 neighborhood, show events; hide / pin / rebuild actions
- Toast notifications, search filter, NER-type filter, rebuild progress bar with ETA
- 51 eval queries (15 new for sprint: disambiguation, no-sonar, sonar-fires, long-conversation, NER-quality, closeness, tier-override, alias-first)



- **Phase 0** (infrastructure): SQLCipher per-user brain.db, schema migrations, JWT, FTS5 setup, lemmatize (pymorphy3), DEK in `~/.dek/<user>.bin`
- **Phase 1** (Layer 1 Native Search): events ingest from TG (608K events for primary user), FTS5 lemmatized index, `/jippy/brain/search`, `/jippy/brain/timeline`
- **Phase 2.1** (Layer 2a Profile) — **TIERED v1 per §17.5.1**:
  - 3 tiers: fact / inference / hypothesis with confidence + evidence + last_confirmed + decay_after_days
  - Tier 1 heuristic (city/lang/active_hours), Tier 2 Cerebras LLM extraction of explicit user statements, Tier 3 Cerebras hypothesis synthesis (cross-field reasoning)
  - Lazy decay: inference→hypothesis→archive on every read
  - Backward-compat migration of legacy flat profiles
  - Endpoints: `GET /profile`, `POST /profile/extract`, `POST /profile/set`, `GET /profile/hypotheses`, `POST /profile/confirm`, `POST /profile/reject`
  - `format_for_prompt()` separates ФАКТЫ vs ВЕРОЯТНО, hypotheses excluded from prompt
- **Phase 2.2** (Layer 2b People): 80 persons extracted, voice samples (10 per person), aliases (canonical + display + first-name + lemma forms). **Parallelised** (8 workers, ThreadPoolExecutor) — 9 min → ~22 sec
- **Phase 2.5** (Layer 2c Knowledge Map): 20499 concepts promoted, 4M edges, 1.8M concept→event links, 131 sec for full backfill (608K events). Stop-words filter still naïve (top concepts include "чтобы", "плиз" — TODO: tighten)
- **Backbone multi-tier fallback**: Cerebras Free → Cerebras Paid → Groq. SSH tunnel via Hetzner Caddy (Vadim → Cerebras Cloud — Vadim IP on CF blocklist). Tier list driven by `JIPPY_BACKBONE_TIERS` env
- **Query rewriter** (Cerebras): expands conversational queries ("блин не могу найти адрес Виталика") → `{search_query, sender_hint, self_only}`. Resolves pronouns via history. ~1.8s p50 latency
- **Multi-turn context**: UI tracks last 8 messages, backend forwards as `messages[]` to Cerebras
- **Day-of-week pre-computation**: `[YYYY-MM-DD HH:MM ВТ]` in event format — fixes LLM calendar arithmetic errors
- **Gender-aware Russian prompts**: profile.gender drives "улетел" vs "улетела"
- **End-to-end verified**: `POST /jippy/brain/ask` with rewriter+sender_filter+L1+Cerebras stream, 4.3s total latency

---

## 🚀 Next Sprint (immediate, 2026-04-29 → next 2 weeks)

Pinned by Yuka 2026-04-29 — все 5 пунктов ниже **must ship в ближайший спринт**.

### S1. Layer 2c rebuild с NER (~2-3 дня) ✅ DONE 2026-04-29
**Backend.** Заменить frequency-based extraction на spaCy NER pipeline.
- Только PERSON / LOC / ORG / PRODUCT / EVENT попадают в концепты
- PoS filter NOUN/PROPN, отсев VERB/PART/ADV
- TF-IDF importance + min_freq ≥ 3
- Aliases extraction (Швеция/Швецию/Швеции/Sweden → один концепт)
- Re-extract на 608K events с новым pipeline
- Eval: top-100 концептов после rebuild не должны содержать стоп-слов
- См. `JIPPY_BACKEND_USER_TODO.md` §11
- Спека: `jippy_architecture_final.md` §17.5.3

### S2. Closeness multi-signal scoring (~1 день) ✅ DONE 2026-04-29
**Backend.** Заменить «top-N by interaction_count» на multi-signal closeness.
- 8 сигналов (см. §17.5.2): reply_latency, self_initiated, emotional_content, bidirectionality, time_of_day, length, diversity, persistence
- `closeness_score` (0..1) + `closeness_tier` (close/warm/acquaint/transactional)
- `user_override` flag — manual UI override не перезаписывается auto-recompute
- См. `JIPPY_BACKEND_USER_TODO.md` §12
- Спека: `jippy_architecture_final.md` §17.5.2

### S3. UI «Кто я» — Profile editor (~2 дня frontend) ✅ DONE 2026-04-29
**Frontend (existing web UI).** 3 секции по tier'ам (Факты / Вероятно / Догадки) с edit, confirm, reject, manual add.
- Все backend endpoints уже готовы: `/profile`, `/profile/extract`, `/profile/set`, `/profile/hypotheses`, `/profile/confirm`, `/profile/reject`
- Spec: `jippy_architecture_final.md` §17.5.1 «UI требование»

### S4. UI «Друзья» — Relational Graph kanban (~2-3 дня frontend) ✅ DONE 2026-04-29
**Frontend (existing web UI).** Kanban 4 колонки (close/warm/acquaint/transactional) с drag-and-drop.
- Drag between columns = manual `closeness_tier` override
- Per-card: voice samples preview, whitelist toggle, edit relationship_type, hide/archive
- Зависит от S2 (нужны closeness_score/tier поля)
- Spec: `jippy_architecture_final.md` §17.5.2 «UI требование»

### S5. UI «Карта идей» — Knowledge Map graph (~3-4 дня frontend) ✅ DONE 2026-04-29
**Frontend (existing web UI).** Force-directed graph через `react-force-graph` или `cytoscape.js`.
- Узел = концепт, размер = importance, цвет = NER-тип
- Действия: hide/delete (user_hidden), pin to core (pinned_to_core), merge duplicates, add edge, search/zoom
- Зависит от S1 (без чистых концептов карта = мусор)
- Spec: `jippy_architecture_final.md` §17.5.3 «UI требование»

### S6. Harness v2 — parallel fetch + adaptive context (~2 дня backend) ✅ DONE 2026-04-29
**Backend.** Заменить regex intent classifier + per-intent assembly на **parallel fetch всех local Layer 1+2 источников** + adaptive context window.

**Без этого S1-S5 не дают полного эффекта** — плохой routing маскирует их (regex классификатор не понимает «Семих» vs «Эйнштейн» vs кличка собаки).

**Подзадачи:**
- **6.1 Parallel fetch.** В `harness.route()` параллельно (asyncio/threadpool) опрашиваются ВСЕ локальные источники: Layer 1 (FTS5), 2a (profile), 2b (people), 2c (concepts). Все находки → в context. ~100ms total для всех.
- **6.2 Adaptive context window.** В backbone call вместо hardcoded last-8 messages — token-budgeted compression: если total ≤ 10K tokens → отдаём всё; иначе → саммари старого через Cerebras + last 2 turns verbatim + current message verbatim.
- **6.3 Layer 0 split.** 0a (backbone parametric — default, бесплатно) + 0b (Sonar web — ТОЛЬКО при temporal markers «сегодня/новости/курс/погода»). Убрать всегда-вызов Sonar, экономим 1-2 сек.
- **6.4 Удалить `_INTENT_PATTERNS` regex** + per-intent `_assemble_*` функции. LLM сам разбирается через semantic context. Кода становится **меньше**, не больше.
- **6.5 Eval queries расширить:**
  - Disambiguation tests: "Семих" → person | "Эйнштейн" → world fact | "Лайка" (если в концептах как pet) → concept
  - Long-conversation test: chat history >10K tokens → ответ всё ещё корректный с компрессией
  - No-regression: текущие S1-S5 queries должны продолжать работать

**Почему не перечил S1-S5:**
- Latency примерно та же (~3.5 сек p50) — local fetches параллельные, дешёвые
- Risk: Cerebras может «теряться» в too much context → mitigation: явные section headers в prompt («=== PROFILE FACTS ===», «=== RELEVANT MEMORIES ===» и т.д.)

**Spec:** `jippy_architecture_final.md` §17.8 (переписан routing tree).

### Что в этом спринте **НЕ делаем**

- Hard rules check в harness — **убран** до Phase Б (когда появится UI для управления). См. §17.8 note.
- iOS port — Phase Б
- LoRA hooked в backend — Phase 4+ (continued)
- Mesh / LoRA Hunt — Phase Б
- Sensors — Phase Б

### Sprint timeline

| Item | Тип | Дни |
|---|---|---|
| S1 Layer 2c NER | Backend | 2-3 |
| S2 Closeness scoring | Backend | 1 |
| S3 Profile UI | Frontend | 2 |
| S4 People UI | Frontend | 2-3 |
| S5 Knowledge Map UI | Frontend | 3-4 |
| S6 Harness v2 | Backend | 2 |

**Backend total:** 5-6 дней. **Frontend total:** 7-9 дней. Параллельно — **спринт 2 недели**, по-прежнему комфортно.

**Backbone в спринте:** все LLM-вызовы продолжают идти через **Cerebras 70B → Cerebras Paid → Groq fallback**. LoRA в backend wiring пока не подключаем — это Phase 4+ с явным switch criteria (см. §17.8 «Backbone roadmap»).

---

## Цель документа

Один файл с актуальным build-order для backend MVP. Когда возникает вопрос «что делать дальше?» или «куда это положить?» — сюда. Не в чаты, не в комменты, не в отдельные .md.

**Правило:** план меняется через явный edit этого файла, не через устные договорённости.

---

## Принципы

1. **End-to-end срез на каждой фазе.** Конец фазы = работающий продукт, не «база данных готова, но ничего не делает».
2. **Defer complexity.** Mesh, Phase Г community FT, L10 ceremony, multi-LoRA composition — не первый MVP.
3. **Mapping в §17.** Каждая фаза имеет точный адрес в спеке. Если фаза не ложится на §17 — либо спека неправильная, либо фаза неправильная. Решаем до старта.
4. **Eval с первого дня.** Без golden queries не понимаем работает или нет.
5. **Privacy с первого дня.** Encryption, не «добавим потом».
6. **Realistic timeline + буфер.** Обещания строим на верхней оценке, не на оптимистичной.

---

## Связь с §17 Memory Architecture

| Слой §17 | Где реализуется в плане |
|---|---|
| Слой 0 (World Knowledge) | Phase 3.1 |
| Слой 1 (Native Search) | Phase 1 |
| Слой 2a (User Profile) | Phase 2.1 |
| Слой 2b (Relational Graph) | Phase 2.2 |
| Слой 2c (Knowledge Map) | **Phase 2.5** (выделена отдельно) |
| Слой 2d (Operational) | Phase 1 (schema) + populate в Phase 3 |
| Слой 2e (Anti-patterns) | Schema в Phase 1, наполнение по факту |
| Слой 3 (LoRA) | Phase 4 |
| Слой 4 (FT) | Не делаем в MVP. Phase Г позже. |
| Harness (orchestration) | Phase 1 (skeleton) → Phase 3 (full) |

---

## Phase 0 — Infrastructure foundations (~3-4 дня)

**Цель:** инфра которая нужна **с первого дня**, чтобы потом не переписывать всё.

### 0.1 Schema migrations skeleton (~2h)

```
backend/migrations/
  001_initial.sql
  002_<future>.sql
backend/app/brain/db.py:
  schema_meta(version INTEGER, applied_at TIMESTAMP)
  apply_migrations() on startup
```

Без этого через 2 месяца боимся менять схему или ломаем существующих юзеров.

### 0.2 Encryption layer (~4h)

- **SQLCipher** для on-disk шифрования brain.db
- Key derivation: user passcode + device-specific salt (PBKDF2 / scrypt)
- Никаких plain-text копий на диске
- rclone backup encrypts на лету (если используется)

**Без этого:** TG история в открытую на любой утечке backup → privacy promise сломан.

### 0.3 Eval harness skeleton (~4h)

```
backend/eval/
  queries.jsonl          # 50-100 golden queries с expected behavior
  run.py                 # запускает все queries, выдаёт report
  reports/<phase>_<date>.json
```

Запускается в конце каждой фазы. Side-by-side «with-X vs without-X» для архитектурных решений (например: «harness on/off», «knowledge_map on/off»).

**Без этого:** через месяц «вроде стало хуже но мы не уверены». Phase 0 lesson из memory: generation > loss, всегда явный eval.

### 0.4 Streaming endpoint contract (~2h)

```python
POST /jippy/ask
Accept: text/event-stream
Body: {question, user_id, stream: true}
Response: SSE stream of {token | event | done}
```

Заглушка пока возвращает echo. Реализация в Phase 3.

**Зачем сейчас:** request-response переделывать на streaming потом — больно. Контракт фиксируем сразу.

### 0.5 Realtime listener wiring (~1 день)

`realtime_listener.py` (Phase D из §15 уже есть) подцепить к новой schema. TG bot/userbot слушает новые сообщения → пишет в `events` table инкрементально.

**Без этого:** Layer 1 = static snapshot, Jippy через неделю устарел.

### Артефакт Phase 0
Пустая, но **правильно настроенная** инфра. Schema versioned, encrypted, eval ready, streaming contract зафиксирован, realtime listener подключён.

---

## Phase 1 — Layer 1 Foundation (~1.5 недели)

**Цель:** конец фазы = можно спросить «что я говорил про X?» и получить ответ из своих сообщений. End-to-end search работает.

### 1.1 Full schema (один SQLite DB на юзера: `backend/storage/users/<uid>/brain.db`)

```sql
-- Layer 1: episodic
events(
  id TEXT PRIMARY KEY,           -- hash(source, source_id, ts)
  ts TIMESTAMP NOT NULL,
  source TEXT NOT NULL,          -- tg | chrome | email | manual | ...
  sender_id TEXT,
  chat_id TEXT,
  raw_text TEXT,
  metadata_json TEXT
);
CREATE INDEX idx_events_ts ON events(ts DESC);
CREATE INDEX idx_events_sender ON events(sender_id);
CREATE INDEX idx_events_chat ON events(chat_id);

CREATE VIRTUAL TABLE events_fts USING fts5(
  text, content=events, content_rowid=rowid
);

-- Layer 2a: profile
user_profile(uid TEXT PRIMARY KEY, profile_json TEXT, updated_at TIMESTAMP);

-- Layer 2b: relational
people(
  id TEXT PRIMARY KEY,
  display_name TEXT,
  canonical_name TEXT,           -- entity resolution target
  relationship TEXT,
  contact_info_json TEXT,
  voice_samples_json TEXT,
  topics_discussed_json TEXT,
  topics_avoided_json TEXT,
  whitelist_for_correction BOOLEAN DEFAULT 0,
  last_interaction TIMESTAMP
);
people_aliases(alias TEXT, person_id TEXT REFERENCES people(id));
people_edges(p1 TEXT, p2 TEXT, edge_type TEXT, strength REAL);

-- Layer 2c: knowledge map
concepts(
  id TEXT PRIMARY KEY,
  name TEXT,
  level INTEGER,                 -- LoRA Hunt mastery level if applicable
  created_at TIMESTAMP
);
-- Embeddings via sqlite-vss virtual table:
CREATE VIRTUAL TABLE concepts_vss USING vss0(
  embedding(384)                 -- mxbai-embed dim
);
concept_edges(c1 TEXT, c2 TEXT, edge_type TEXT, strength REAL, user_explicit BOOLEAN);
concept_to_events(concept_id TEXT, event_id TEXT);

-- Layer 2d: operational
corrections(id TEXT PRIMARY KEY, ts TIMESTAMP, query TEXT, jippy_answer TEXT, user_correction TEXT, priority INTEGER);
knowledge_cards(id TEXT PRIMARY KEY, source_pubkey TEXT, topic TEXT, insight TEXT, provenance_json TEXT, utility_score REAL, signed_at TIMESTAMP);
lora_hunt_tags(tag TEXT PRIMARY KEY, level INTEGER, badges_json TEXT, public_visible BOOLEAN);
open_questions(id TEXT PRIMARY KEY, theme TEXT, first_seen TIMESTAMP, last_revisited TIMESTAMP, count INTEGER);

-- Layer 2e: anti-patterns
antipatterns(id TEXT PRIMARY KEY, type TEXT, content TEXT, scope TEXT);

-- Layer 4: harness logs
hard_rules(id TEXT PRIMARY KEY, rule TEXT, scope TEXT, active BOOLEAN);
inference_log(id TEXT PRIMARY KEY, ts TIMESTAMP, query TEXT, route TEXT, used_layers_json TEXT, latency_ms INTEGER);
```

**Migration 001** содержит всю эту схему. Будущие изменения = новые migrations.

### 1.2 Layer 1 ingestion (`backend/app/brain/ingest.py`)

- Прогоняет `reply_pairs.jsonl` + `chats/*.jsonl` → events table + FTS5
- Idempotent по `event_id = hash(source, source_id, ts)`
- Edited messages: latest-wins, старая версия в metadata_json как `prev_versions[]`
- Encrypted media (фото, голос) — пока пропускаем, decode pipeline в Phase Б

### 1.3 Layer 1 search API (`backend/app/routers/brain.py`)

```
GET /jippy/brain/search
  ?q=<query>
  &k=20
  &sender=<id>           # filter
  &chat=<id>             # filter
  &since=<ts>&until=<ts> # date range
  &source=<tg|chrome|...>
  → BM25 (FTS5) + ts ranking → top-k events

GET /jippy/brain/event/:id → full event
GET /jippy/brain/timeline?sender=&from=&to= → chronological view
```

Pagination: cursor-based, не offset (для больших историй).

### 1.4 Privacy filter

Какие события из FTS5 исключены:
- Помеченные `private=true` в metadata
- Из контактов в private list
- Phase 0 encryption + per-event sensitivity flag

### 1.5 Harness skeleton (`backend/app/brain/harness.py`)

Заглушка которая просто роутит в Layer 1 search:
```python
def route(query, user_id):
    return {
        "route": "layer1_search",
        "results": layer1.search(query, user_id),
        "context_for_llm": format_episodes(results)
    }
```

Расширим в Phase 3.

### 1.6 Eval queries v1

Минимум 30 golden queries в `eval/queries.jsonl`:
- 10 episodic ("что я говорил Васе про вино")
- 10 search-by-attribute ("сообщения от Х в апреле")
- 10 negative cases (queries которые НЕ должны вернуть результаты)

Run в конце Phase 1 → baseline metrics.

### 1.7 Failure modes (degraded behavior)

- Empty result → API 200 с `{results: [], reason: "no_match"}`, не 404
- FTS5 unavailable → fallback на LIKE-search (медленно но работает)
- DB locked → retry with exponential backoff, max 3

### Артефакт Phase 1
**Работающий поиск по истории юзера, end-to-end.** Это уже полезный продукт сам по себе (поиск по своим перепискам). Eval baseline зафиксирован.

---

## Phase 2 — Layer 2a/2b Extraction (~1 неделя) ✅ DONE 2026-04-29

**Цель:** конец фазы = базы заполнены извлечёнными знаниями. Можно спросить «что я думаю про X?» и получить структурированный профиль/мнение.

### 2.1 Layer 2a — Profile extraction (`backend/app/brain/extract_profile.py`) ✅

**ВАЖНО:** реализация ушла дальше плановой flat-схемы и теперь следует **§17.5.1 (tiered, hypothesis-driven)** из `jippy_architecture_final.md`. Описание ниже сохранено как историческое; актуальная схема — в §17.5.1 спеки.

**Pipeline:**
```
events table → chunking strategy → Qwen 0.8B (local, never Sonar) →
  structured JSON → merge с existing profile → user_profile table
```

**Chunking strategy:** per-thread (1 conversation = 1 chunk), max 8K tokens. Если переборщит — split по dialog turns.

**Output schema:**
```json
{
  "home_address": "...",
  "work_address": "...",
  "phase_of_life": "...",
  "current_projects": [...],
  "values_inferred": [...],
  "languages": [...],
  "routine_patterns": {...},
  "preferred_aesthetic": "...",
  "extraction_metadata": {
    "last_run": "...",
    "events_processed": N,
    "confidence": 0..1
  }
}
```

**Merge policy** (критично, не упустить):
- Новый факт принимается если: подтверждён ≥2 раза в новом батче ИЛИ явно сказан недавно (last_30_days)
- Старый факт сохраняется если: новый батч его НЕ противоречит явно
- Конфликт: оба факта в `profile_json.history[]`, current = более свежий + более частый

**Privacy:** Qwen на сервере (не Sonar) — данные не покидают нашу инфру. Юзер может опт-аут per category.

**Schedule:** initial run on backfill, потом раз в 7 дней или при `events_since_last_run > 1000`.

### 2.2 Layer 2b — Relational graph extraction (`backend/app/brain/extract_people.py`) ✅

Реализовано: 80 persons, voice samples + topics extraction параллелизована (8 workers, ThreadPoolExecutor с per-thread SQLCipher conn). Aliases расширены (canonical + display + first-name + lemma) — теперь "Виталик" резолвится без полного имени. Время: ~22 сек на 80 персон (раньше ~9 мин).

**Pipeline:**
```
events table → group by counterparty → metrics (frequency, duration, topics) →
  Qwen extracts relationship_type → people table
```

**Entity resolution** (важно):
- Same person с разными display names: «Вадим» / «Vadim» / «Вадя» → один canonical_name
- Используем: phone number > email > display_name fuzzy match (Levenshtein < 3)
- Aliases стораджатся в `people_aliases` для будущего поиска

**Voice samples extraction:**
- Stratified по топикам и времени (не просто N most recent)
- Для top-3 ближайших: 20 пар, для остальных: 5 пар
- Используются как few-shot в Phase 4 multi-voice C-tier

**Edges (people_edges):**
- Если два контакта часто встречаются в одном чате → edge `co_present`
- Если один пишет про другого → edge `mentions`
- v1 простая логика, не graph theory

### 2.3 Layer 2d/2e — schema only

Schema готова из Phase 1.1, наполнение **по факту использования**:
- Corrections — UX hook в Phase 3 (юзер пишет «нет, на самом деле X»)
- Knowledge cards — после §16 mesh wiring (Phase Б)
- Anti-patterns — extracted from explicit user statements + behavioral patterns в Phase Б
- Open questions — clustering recurring themes в Phase Б

В MVP пустые таблицы — это нормально. Заполняются юзером в процессе использования.

### 2.4 Eval queries v2

+20 golden queries:
- "кто такой <person>" → должен дать relationship + topics
- "что я думаю про <topic>" → должен дать profile-based opinion
- "когда я последний раз говорил с <person>"

### Артефакт Phase 2
Структурированный JSON-портрет юзера + социальный граф. Можно отвечать на «who/what/when» queries без хождения в LoRA.

---

## Phase 2.5 — Layer 2c Knowledge Map (~2-2.5 недели) ✅ v1 DONE 2026-04-29

Реализован v1 backfill: 608K events → 248K candidates → 20499 concepts promoted, 4M edges, 1.8M concept→event links за 131 секунду. Embeddings пока пустые (`embeddings_real: false`). Stop-words фильтр наивный (топ-концепты включают "чтобы", "плиз", "было") — нужна доработка перед production. Endpoints `/jippy/brain/concepts`, `/jippy/brain/concept/{name}` живые.



**Цель:** топология ума юзера как граф. Самая ценная и **самая сложная** часть. Поэтому отдельная фаза, не подзадача.

**Scope v1** (что реалистично за 2-2.5 недели):
- Flat triple extraction (entity, relation, entity)
- Embeddings концептов через mxbai-embed
- Simple co-reference resolution (rule-based + LLM fallback на сложных случаях)
- Pointer-back в Layer 1 (`concept_to_events`)
- Graph query API (depth=1,2 traversal)

**Scope Phase Б** (deferred):
- Community detection (Leiden algorithm)
- Hierarchical summarization (Microsoft GraphRAG full)
- Multi-hop reasoning
- Graph visualization UI

### 2.5.1 Pipeline

```
nightly batch (Qwen idle on charger / scheduled cron):
  events_since_last_run →
  Qwen extracts (entity, relation, entity) triples →
  co-reference resolution (rule-based first, LLM fallback) →
  entity merge (new vs existing concepts) →
  embed new concepts via mxbai-embed →
  store in concepts + concept_edges + concepts_vss →
  link к events в concept_to_events
```

### 2.5.2 Entity merge strategy

- Exact name match → merge
- Fuzzy match (Levenshtein < 3 + cosine sim > 0.85) → propose merge для review
- New concept → create

Без entity merge граф быстро превращается в дубликаты («GPT-4», «gpt 4», «GPT4» — три узла).

### 2.5.3 Co-reference resolution

**v1 простая:**
- Rule-based: pronouns в одном dialog turn → previous proper noun
- LLM fallback: если rule-based unsure, отправляем мини-batch в Qwen с промптом «resolve pronouns»

**Phase Б:**
- Full neural co-ref (allennlp / huggingface coref models)

### 2.5.4 Query API

```
GET /jippy/brain/concept/:name
  → { concept, level, edges: [{to, type, strength}], events: [...] }

GET /jippy/brain/concept/:name/neighbors?depth=2
  → subgraph centered on concept

GET /jippy/brain/concept-search?q=<semantic query>
  → top-k concepts by embedding similarity
```

### 2.5.5 Eval

Specifically для knowledge map:
- 20 queries «что у меня связано с X» → expected concepts list (manually labeled на subset Юкиных данных)
- Side-by-side: harness with knowledge map vs without

### Артефакт Phase 2.5
**Граф концептов юзера** с pointer-back в исходники. Frontier этого by design не имеет — у них один граф для всех.

---

## Phase 3 — Inference Loop (~1.5 недели)

**Цель:** работающий end-to-end inference. Юзер задаёт вопрос → harness роутит по слоям → ассемблирует hint → вызывает backbone → возвращает ответ (streaming).

### 3.1 Layer 0 wiring

**0a:** уже есть Qwen on-device (для разработки — server-side fallback на server Qwen)

**0b:** Sonar API client (live на gpu.social per memory) — обёртка с timeout/retry/error handling

**0c:** defer to Phase Б

### 3.2 Backbone API (`backend/app/routers/inference.py` — расширить)

```
POST /jippy/ask
Accept: text/event-stream
Body: {question, user_id, voice_target?, conversation_id?}

Response: SSE stream:
  event: route        data: {layers_used: [...]}
  event: token        data: {text: "..."}
  event: token        data: {text: "..."}
  ...
  event: done         data: {total_latency_ms: N}
```

Backbone choice:
- Server-side: Qwen 7B / 32B (бóльшая модель чем on-device)
- Phone-side (Phase 4): Qwen 0.8B + LoRA

**Same harness, different backbones.** Это контракт.

### 3.3 Harness routing — full (`backend/app/brain/harness.py`)

Реализует §17.8 routing tree.

**Intent classifier:** LLM-based via Qwen 0.8B

```python
def classify_intent(query) -> RouteDecision:
    # Prompt small Qwen с few-shot examples (~20 examples)
    # → returns one of:
    #   "episodic" | "semantic_self" | "person" | "world_fact" | "generation"
    # Latency budget: 200ms
```

Heuristic shortcuts для очевидных случаев (regex для дат, названий) → skip LLM call для скорости.

**Hint assembly:**
```python
def assemble_hint(intent, query, user_id):
    if intent == "episodic":
        return {"episodes": layer1.search(query)}
    elif intent == "semantic_self":
        return {
            "profile": layer2a.get(user_id),
            "concepts": layer2c.query(extract_concepts(query)),
            "antipatterns": layer2e.relevant(query)
        }
    elif intent == "person":
        return {"person": layer2b.get(extracted_name)}
    elif intent == "world_fact":
        return {"web": layer0b.search(query)} if online else {"backbone": "use_parametric"}
    elif intent == "generation":
        return {
            "voice_samples": layer2b.voice_for(voice_target),
            "concepts": layer2c.query(...),
            "antipatterns": layer2e.relevant(query)
        }
```

### 3.4 LLM call

```python
def call_backbone(hint, query):
    prompt = format_prompt(hint, query)  # template per intent type
    return backbone.generate_stream(prompt)
```

Prompt templates per intent в `prompts/` directory.

### 3.5 Failure modes

- Layer 1 empty → "я тебя ещё плохо знаю" + Layer 0 fallback
- Layer 2 пуст → degraded mode без personalization
- Backbone unavailable → cached response if exists, иначе error
- Sonar API down → backbone parametric only
- All systems down → graceful error, не crash

### 3.6 Inference log

Каждый call логируется в `inference_log`:
- query, route, layers_used, latency_ms
- Используется для eval, debug, postmortem

### 3.7 Eval queries v3

+30 queries покрывающих все intent types. Specifically:
- A/B comparison: harness on vs off → должно быть statistically significant improvement
- A/B: knowledge_map on vs off → expected improvement on "что я думаю про X" queries
- A/B: with-Layer-2 vs without → expected improvement on personal queries

### Артефакт Phase 3
**Работающий inference сервер.** Можно тестить «было лучше с/без harness», «было ли лучше с Layer 2c knowledge map». Stream-based UX.

---

## Phase 3+ — Multi-step Agent Loop (Hermes adoption, ~1-2 дня)

**Цель:** добавить multi-step reasoning loop поверх существующего harness'а. До этого момента Phase 3 делает single-shot routing → tool/answer. Phase 3+ позволяет N-step agent loops (think → call tool → observe → think → answer).

**Источник:** [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) (MIT license).

**Подход:** **cherry-pick ~700 LOC, не форкать, не зависеть.** Vendor specific files в `backend/app/agent/` с notice `# Adapted from NousResearch/hermes-agent (MIT)`.

### Что вендорим (с путями в их репо)

| Из | LOC | Куда у нас | Зачем |
|---|---|---|---|
| `environments/agent_loop.py` | ~534 | `backend/app/agent/agent_loop.py` | Multi-step loop, AgentResult/ToolError dataclasses, reasoning_content extraction |
| `environments/tool_call_parsers/hermes_parser.py` | ~76 | `backend/app/agent/parsers/hermes.py` | Канонический `<tool_call>{json}</tool_call>` формат |
| `environments/tool_call_parsers/qwen_parser.py` | ~50 | `backend/app/agent/parsers/qwen.py` | Qwen 0.8B уже эмитит этот формат нативно |
| `environments/tool_call_parsers/qwen3_coder_parser.py` | ~80 | `backend/app/agent/parsers/qwen3_coder.py` | Если переходим на coder-вариант |
| `tools/session_search_tool.py` (lines 319-520) | — | паттерн для `app/brain/layer1.py` enhancement | Parent-chain dedup, role filter |
| `environments/hermes_base_env.py` | — | паттерн для `backend/eval/run.py` enhancement | Reward-fn-with-tool-access для LoRA evals |

### Что НЕ берём (документируем явно)

- `run_agent.py` (13K LOC monolith) — coupled с 60-param init, читать только для идей
- `agent/anthropic_adapter.py` / `bedrock_adapter.py` / `gemini_*` — у нас Qwen-only
- Cloud backends (Modal/Daytona/Singularity) — мы edge-first
- `gateway/` (Telegram/Discord/Slack/...) — вне scope MVP
- Honcho dialectic user modeling — у нас свой Phase 3 relational layer (Layer 2b)
- `prompt_builder.py` (1122 LOC) — слишком coupled

### Интеграция в наш harness

```python
# До (Phase 3): single-shot
def route(user_id, query, voice_target=None) -> RouteResult:
    intent = classify_intent(query)
    hint = assemble_hint(intent, query, user_id)
    return RouteResult(intent, hint)

# После (Phase 3+): single-shot OR multi-step
def route(user_id, query, voice_target=None, multi_step=False) -> RouteResult:
    intent = classify_intent(query)
    hint = assemble_hint(intent, query, user_id)
    if not multi_step or intent in ("episodic", "person"):
        return RouteResult(intent, hint)  # single-shot path
    # multi-step: agent_loop с нашими tools (search, concept, profile, etc.)
    return run_agent_loop(query, hint, tools=brain_tool_registry, max_turns=8)
```

Тулзы в registry — наши existing layer1.search / extract_concepts.get_concept / etc.

### Eval

Specifically для multi-step:
- Queries требующие multi-hop reasoning ("кто из моих контактов сейчас в Тбилиси и работает в healthcare")
- Side-by-side: single-shot vs multi-step

### Failure modes

- max_turns hit → возвращаем best-effort answer + log
- Tool error → propagate в trajectory, agent может try alternative
- Циклы (loop detection) → break + log

### Артефакт Phase 3+
Harness умеет multi-step reasoning через canonical Hermes format. Tool-calls Qwen работают zero-friction (он уже эмитит этот формат). Vendoring without dependency — мы независимы от Hermes upstream.

---

## Phase 4 — LoRA integration (~3-4 дня)

**Цель:** наш LoRA встроен как Layer 3. Стиль, не факты.

### 4.1 LoRA wiring (server-side first)

- Phone-side inference loads adapter (deferred to phone port)
- Server-side: backbone + LoRA для тестирования стиля
- Harness формирует prompt с context из Layer 1+2, LoRA генерит в стиле юзера

### 4.2 Multi-voice (минимум — variant A)

**Speaker token conditioning:**
- В обучающие данные префикс `<to:contact_id>...`
- На инференсе harness вставляет `<to:vadim>` если voice_target=vadim
- Один LoRA, много voices

**Требует re-training jippy v1 → v2** с speaker tokens. Phase 0 lesson: generation > loss, save_every=200, eval включает generation samples.

### 4.3 Multi-voice variant C (RAG few-shot)

Для редких контактов где нет re-training capacity:
- Voice samples из 2b в prompt как few-shot examples
- Это работает поверх любого LoRA, не требует re-training

### 4.4 Variant B (per-relationship мини-LoRA)

**Defer to Phase Б.** Top 3-5 ближайших получают свой ~5MB LoRA, активируется через Arrow routing.

### 4.5 Eval

- Voice match metric: насколько ответы похожи на реальный voice юзера (compare against held-out test set)
- Side-by-side: with vs without LoRA, with vs without speaker tokens

### Артефакт Phase 4
**Полный stack v1.** Backbone + LoRA + 5 слоёв памяти + harness routing + multi-voice. Можно показать as proof-of-concept.

---

## Что НЕ делаем в MVP

| Feature | Когда |
|---|---|
| Phone port (iOS/Android) | После server backend стабилен |
| Mesh / LoRA Hunt (§16) | Phase Б |
| Sensor pipeline (§11) | Phase Б |
| Tool use (§10) | Phase Б |
| Per-relationship мини-LoRA (B-tier) | Phase Б |
| Knowledge Map community detection | Phase Б |
| Federated community FT (Phase Г) | Year 2+ |
| L10 personal FT ceremony | Year 3+ |
| Curated cache (Layer 0c) | Phase Б |
| TTT L2/L3 (§12.3, §12.4) | Phase В+ |

---

## Realistic Timeline

| Фаза | Содержание | Дни |
|---|---|---|
| 0 | Infra foundations | 3-4 |
| 1 | Layer 1 + extended | 7-10 |
| 2.1 | Profile extraction | 3 |
| 2.2 | Relational graph | 4 |
| 2.5 | Knowledge Map | 10-13 |
| 3 | Harness + backbone | 8-10 |
| 3+ | Hermes agent_loop adoption | 1-2 |
| 4 | LoRA wiring | 3-4 |

**Итого: 39-50 дней = 5.5-7 недель.**

Обещания строим на **верхней оценке (7 недель)**, не на оптимистичной. Если успели быстрее — buffer для polish и eval.

---

## Risk Register

| Риск | Severity | Mitigation |
|---|---|---|
| GraphRAG implementation overrun | 🔴 high | Выделена отдельная Phase 2.5 с реалистичным scope |
| Entity resolution edge cases | 🟡 medium | Fuzzy match thresholds tunable, fallback на LLM |
| Profile merge conflicts | 🟡 medium | Явная merge policy + history в profile_json |
| Intent classifier latency | 🟡 medium | Heuristic shortcuts + 200ms budget + caching |
| Schema migrations later | 🟡 medium | Phase 0.1 закрывает превентивно |
| Privacy leak | 🔴 high | Phase 0.2 SQLCipher с первого дня |
| Eval drift / false positives | 🟡 medium | Phase 0.3 eval skeleton + per-phase regression |
| Backbone availability | 🟡 medium | Failure mode handling в Phase 3.5 |
| Re-training LoRA для multi-voice | 🟢 low | Существующий pipeline есть (jippy v1) |

---

## Eval Queries — структура файла

`backend/eval/queries.jsonl` — JSONL формат, по одному query в строке:

```json
{
  "id": "ep_001",
  "category": "episodic",
  "query": "что я говорил Васе про вино",
  "expected": {
    "min_results": 1,
    "must_contain_terms": ["вино"],
    "must_be_from_sender": "Vasya"
  },
  "phase_added": 1
}
```

Categories:
- `episodic` (Layer 1)
- `profile` (Layer 2a)
- `person` (Layer 2b)
- `concept` (Layer 2c)
- `world_fact` (Layer 0)
- `generation` (Layer 3)
- `negative` (must NOT return results)
- `failure_mode` (degraded behavior expected)

---

## Что после MVP

После того как этот план выполнен и MVP стабилен:

1. **Phone port** — iOS first (есть on-device LoRA training validated в §15)
2. **Phase Б features** — sensors, tool use, mesh, knowledge map advanced
3. **LoRA Hunt v0** — opt-in карта без BLE
4. **TTT Level 2** — session-LoRA implementation

Это всё в `FRONTIER_ATTACK.md` Sequencing разделах.

---

## Изменение этого плана

**Правило:** план меняется через explicit edit + commit + краткое summary в commit message. Не через устные договорённости.

**Когда обновлять:**
- В конце каждой фазы (что получилось vs план)
- При изменении scope (добавили/убрали фичу)
- При значимом overrun (>20% времени) — пересчитать оставшиеся фазы
- При архитектурном изменении в `jippy_architecture_final.md` — sync sections

Last revision: 2026-04-28 (initial)
