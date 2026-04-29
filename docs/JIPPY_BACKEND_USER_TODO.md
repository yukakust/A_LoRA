# Jippy Backend — что нужно от Юки

**Статус:** живой (2026-04-28)
**Контекст:** этот файл собирает всё что я (Claude) не могу сделать сам и нужно от тебя — закрытые секреты, провижионинг серверов, ручные оценки результатов, бизнес-решения.

Я довёл backend до полностью рабочего состояния по всем фазам IMPLEMENTATION_PLAN.md (0, 1, 2, 2.5, 3, 4) с stub-режимом backbone и LoRA уже подключён. Остальное — ниже.

---

## P0 — Блокирует прод-готовность

### 1. Backbone hosting (Phase 3.1)
**Что:** backend сейчас вызывает backbone через `JIPPY_BACKBONE` env:
- `stub` (default): эхо-режим, нужен только для dev и тестов
- `openai`: любой OpenAI-compatible endpoint (Vadim's server, gpu.social, Anthropic API gateway)
- `mlx`: локальный mlx_lm Qwen + LoRA (для фоны/планшеты)

**Что нужно:** выбрать backbone и сообщить URL + key. Варианты:
- **Vadim's Qwen 7B** на нашем сервере (CPU сейчас, GPU скоро)
- **gpu.social /api** (он уже есть в memory, есть Sonar там)
- **Anthropic Claude Sonnet/Haiku** через api.anthropic.com (платно, но качество)

**Решение пишется в `.env`:**
```
JIPPY_BACKBONE=openai
JIPPY_BACKBONE_URL=https://...
JIPPY_BACKBONE_KEY=...
JIPPY_BACKBONE_MODEL=qwen2.5-7b-instruct
```

**Пока этого нет — `/jippy/brain/ask` работает в stub-режиме (echo).**

### 2. Eval golden queries — ground truth (Phase 1.6)
**Что:** я написал 28 golden queries в `backend/eval/queries.jsonl` чтобы прогонять регрессию между фазами. Часть из них умозрительные — я не знаю что РЕАЛЬНО есть в твоей TG-истории.

**Что нужно от тебя:** прогнать `python -m backend.eval.run --user-id u_2ca9a174b77dcb7b --phase 1`, посмотреть какие episodic-queries failед, и:
- либо переформулировать query (например "разговор про вену" → "разговоры о Вене")
- либо удалить если эта тема правда не обсуждалась
- добавить 10-20 СВОИХ golden queries по которым ты сама знаешь правильный ответ

**Формат:**
```json
{"id": "ep_X", "category": "episodic", "phase_added": 1,
 "query": "что я говорил Игорю про rates",
 "expected": {"min_results": 1, "must_contain_terms": ["rate"]}}
```

### 3. SSO/JWT secret в prod
**Что:** `JWT_SECRET` сейчас дефолтный "dev-secret-change-me". В prod нужен нормальный секрет в `backend/app/config.py` или env.

```
JWT_SECRET=<32+ random bytes>
```

### 4. Sonar API key (для Layer 0b world facts)
**Что:** Layer 0b — web search для свежих world facts ("новости", "погода"). Сейчас выключен (route → backbone parametric). Чтобы включить:

```
JIPPY_SONAR_KEY=<sonar key from gpu.social>
```

После этого harness `_assemble_world_fact` будет звать Sonar (TODO: implement Sonar client wrapper).

---

## P1 — улучшает качество, не блокирует

### 5. Профильное LLM-обогащение (Phase 2.1 Tier 2)
**Что:** Tier 1 profile уже работает (heuristic — топ-города, языки, активные часы). Tier 2 — LLM прогоняется по образцу твоих сообщений и извлекает:
- `current_projects`, `values_inferred`, `interests`, `phase_of_life`

**Что нужно:** прогнать вызов с реальным LLM. В коде:
```python
from app.brain.extract_profile import extract_profile_run
def my_llm(prompt): return ...  # call any OpenAI-compatible / mlx-lm
extract_profile_run(user_id, llm=my_llm)
```

Мы зашьём это в `/jippy/brain/profile/extract` когда выберется backbone (см. P0.1).

### 6. Real concept embeddings (Phase 2.5)
**Что:** концепты сейчас имеют zero-vector embeddings (stub). Для семантического поиска "найди концепты похожие на X" нужны настоящие embeddings.

**Решение:** установить `sentence-transformers` + модель `mixedbread-ai/mxbai-embed-large-v1` локально или подключить server endpoint (Vadim уже хостит embedding модели).

**Команда чтобы пересчитать:**
```python
extract_concepts_run(user_id, use_real_embeddings=True)
```

### 7. Hard rules — ручной набор
**Что:** harness проверяет hard_rules table перед routing. Сейчас пусто. Чтобы добавить что-то типа "никогда не показывай адреса посторонним":

```sql
INSERT INTO hard_rules(id, rule, scope, active) VALUES
  ('hr_001', 'адрес дома', 'unknown_recipient', 1),
  ('hr_002', 'банковские данные', 'all', 1);
```

UI hooks в Phase Б.

### 8. Whitelist для §12.6 conservative weight updates
**Что:** при extract_people все люди получают `whitelist_for_correction = 0`. Это поле определяет кто из close circle может через знание (b) влиять на твои веса. Тебе нужно явно отметить 20-50 человек как "trusted" — обычно через UI, но пока:

```sql
UPDATE people SET whitelist_for_correction = 1 WHERE id IN (...);
```

UI checkbox в Phase Б.

### 9. Hermes agent_loop adoption (Phase 3+, ~1-2 дня)
**Что:** vendor ~700 LOC из NousResearch/hermes-agent (MIT) для multi-step reasoning. Подробности в `IMPLEMENTATION_PLAN.md` Phase 3+.

**Что взять:**
- `environments/agent_loop.py` (~534 LOC) → `backend/app/agent/agent_loop.py`
- `environments/tool_call_parsers/{hermes,qwen,qwen3_coder}.py` → `backend/app/agent/parsers/`
- Паттерны из `tools/session_search_tool.py` (parent-chain dedup) для enhancement `layer1.py`
- Паттерны из `environments/hermes_base_env.py` для enhancement `backend/eval/run.py`

**Что НЕ брать:**
- `run_agent.py` (13K LOC monolith — слишком coupled)
- Cloud adapters (anthropic/bedrock/gemini) — у нас Qwen-only
- Cloud backends (Modal/Daytona/Singularity) — мы edge-first
- `gateway/` (мессенджеры) — out of scope MVP
- Honcho user modeling — конфликт с privacy-first, у нас свой Layer 2b

**Notice в каждом vendored файле:**
```python
# Adapted from NousResearch/hermes-agent (MIT License)
# Source: https://github.com/NousResearch/hermes-agent
# Copyright 2025 Nous Research
```

**Зачем именно сейчас:** Qwen 0.8B уже эмитит `<tool_call>{json}</tool_call>` нативно. Адаптация их парсера = zero-friction. agent_loop.py закрывает наш единственный пробел в harness — multi-step reasoning. Не fork, не dependency — vendor.

### 10. ✅ Tiered User Profile redesign (§17.5.1) — **DONE**
Реализовано в коммите `0ab558e` jippy backend. Полная схема + extraction Tier 1/2/3 + active testing endpoints + decay (lazy on read). Спека: `jippy_architecture_final.md` §17.5.1. UI ещё не сделан — см. §13.1 в этом документе.

### 11. ✅ DONE — Layer 2c rebuild с NER (§17.5.3, ~2-3 дня)
*Реализовано 2026-04-29: spaCy ru/en lang-aware, NER+PoS pipeline, lemma-canonical concepts, surface-form aliases, TF-IDF importance, background rebuild job + status endpoint, user curation hooks (hide/pin/merge/edge).*


**Что:** заменить frequency-based concept extraction на spaCy NER pipeline. Текущая v1 — degraded mode (топ концептов = «плиз», «чтобы», «было»).

**Полная спека:** `docs/jippy_architecture_final.md` §17.5.3 «Extraction — обязательно через NER».

**Реализация:**

```python
# 1. Установить spaCy + multilingual model
pip install spacy
python -m spacy download xx_ent_wiki_sm    # multilingual NER
# или ru_core_news_sm для русского, en_core_web_sm для английского

# 2. В extract_concepts.py заменить токенизацию + frequency-фильтр на:
import spacy
nlp = spacy.load("xx_ent_wiki_sm")

def extract_concepts_from_event(text):
    doc = nlp(text)
    concepts = []
    # NER: только эти типы
    for ent in doc.ents:
        if ent.label_ in {"PERSON", "LOC", "GPE", "ORG", "PRODUCT", "EVENT", "WORK_OF_ART"}:
            concepts.append({"name": ent.text, "type": ent.label_})
    # PoS fallback: NOUN/PROPN которые НЕ stopword
    for token in doc:
        if token.pos_ in {"NOUN", "PROPN"} and token.text.lower() not in STOPWORDS:
            concepts.append({"name": token.lemma_, "type": "noun"})
    return concepts
```

**Aliases extraction:**
- Швеция / Швецию / Швеции / Sweden / SW → один canonical concept
- Используется fuzzy match (Levenshtein < 3) + lemma normalization
- Хранятся в существующей `concept_aliases` table

**Re-extract** на всех 608K events с новым pipeline. Старые концепты (с мусором) — wipe + rebuild.

**Eval тесты:**
- Top-100 концептов после rebuild не должны содержать стоп-слов («плиз», «чтобы», и т.д.)
- Концепт «Швеция» должен существовать с aliases
- Запрос `GET /jippy/brain/concept/Швеция?depth=1` → 200 OK + linked events

**Зависит от:** ничто блокирующее. Можно делать параллельно с UI работой.

---

### 12. ✅ DONE — Closeness multi-signal scoring (§17.5.2, ~1 день)
*Реализовано 2026-04-29: 8-signal weighted score (latency/self-init/emotional/bidirect/time/length/diversity/persistence), tier mapping close/warm/acquaint/transactional, user_override flag protected from auto-recompute, override endpoints, with_signals breakdown.*


**Что:** заменить «top-N by interaction_count» на multi-signal closeness в `extract_people.py`.

**Полная спека:** `docs/jippy_architecture_final.md` §17.5.2 «Closeness multi-signal score».

**Schema delta:**
```sql
ALTER TABLE people ADD COLUMN closeness_score REAL DEFAULT 0.0;
ALTER TABLE people ADD COLUMN closeness_tier TEXT DEFAULT 'acquaint';
ALTER TABLE people ADD COLUMN user_override INTEGER DEFAULT 0;
```

**Расчёт score:**
```python
def compute_closeness(user_id, person_id):
    events = layer1.get_events_with_person(user_id, person_id)

    # 8 signals (нормализованы 0..1):
    reply_latency = score_from_avg_latency(events)        # быстрые = высокий
    self_initiated = ratio_user_started(events)
    emotional = nlp_emotional_content_ratio(events)
    bidirect = bidirectionality_score(events)
    time_personal = ratio_outside_work_hours(events)
    avg_len = normalize(avg_message_length(events))
    diversity = topic_diversity_score(events)
    persistence = months_active(events) / 60.0

    weights = [.15, .15, .20, .15, .10, .05, .05, .15]   # tunable
    signals = [reply_latency, self_initiated, emotional, bidirect,
               time_personal, avg_len, diversity, persistence]
    return sum(w*s for w, s in zip(weights, signals))

def tier_from_score(s):
    if s >= 0.75: return "close"
    if s >= 0.50: return "warm"
    if s >= 0.25: return "acquaint"
    return "transactional"
```

**User override:** при manual change через UI → `user_override=1`. Auto re-compute **не перезаписывает** override'нутые.

**Endpoints добавить:**
- `POST /jippy/brain/people/{id}/tier` — manual override (для UI drag-and-drop)
- `GET /jippy/brain/people?tier=close` — фильтр по tier

**Зависит от:** ничто блокирующее. **Блокирует:** S4 People UI (нужны поля).

---

### 13. ✅ DONE — Frontend UI: Profile / People / Knowledge Map (~7-9 дней frontend)
*Реализовано 2026-04-29 в `backend/app/static/jippy_chat.html` (single-file v1, как договаривались — Chrome ext / standalone web в Phase Б): tab nav + Profile (3 tier sections, edit/confirm/reject/manual add) + People (4-column kanban с DnD = manual tier override, click → 8-signal breakdown) + Map (cytoscape force-directed, NER colors, importance sizing, click expand neighbors, hide/pin/rebuild с progress).*


**Что:** добавить 3 visualization tab'а в существующий Web UI (Chrome ext / web).

**Backend полностью готов** — все endpoints существуют.

#### 13.1 Profile UI «Кто я» (~2 дня)
3 секции (Факты / Вероятно / Догадки), edit/confirm/reject/manual add.
**Спека:** `jippy_architecture_final.md` §17.5.1 «UI требование».
**API:** `/profile`, `/profile/extract`, `/profile/set`, `/profile/hypotheses`, `/profile/confirm`, `/profile/reject`.

#### 13.2 People UI «Друзья» Kanban (~2-3 дня)
4 колонки (close/warm/acquaint/transactional) с drag-and-drop = manual tier override.
Per-card: voice samples preview, whitelist toggle, edit relationship_type, hide/archive.
**Спека:** `jippy_architecture_final.md` §17.5.2 «UI требование».
**API:** `/people?tier=...`, `/person/:id`, `/people/:id/tier` (новый, см. §12).
**Зависит от:** §12 backend (closeness_tier поле должно быть).

#### 13.3 Knowledge Map UI «Карта идей» (~3-4 дня)
Force-directed graph через `react-force-graph` или `cytoscape.js`.
- Узел = концепт, размер = importance, цвет = NER-тип
- Actions: hide/delete (user_hidden), pin to core (pinned_to_core), merge duplicates, add edge, search/zoom
**Спека:** `jippy_architecture_final.md` §17.5.3 «UI требование».
**API:** `/concept/:name?depth=N`, `/concepts?q=...`, новые endpoints для hide/pin/merge нужно добавить.
**Зависит от:** §11 backend (без чистых концептов карта = мусор).

**Стек предложение:** React + react-force-graph + tailwind. Адаптировать под текущий extension UI.

---

### 15. ✅ DONE — Harness v2: parallel fetch + adaptive context (~2 дня backend)
*Реализовано 2026-04-29: ThreadPoolExecutor parallel fetch (Layer 1+2a/2b/2c/2e), conditional Layer 0b на temporal markers, adaptive char-budget compression (30K) с Cerebras summary, NER-based query name extraction, lemma-based concept resolution, единый `_build_system_v2` + `_format_context_with_sections`. Удалены regex intent classifier + 5 per-intent assemblers. End-to-end p50 ~1s (с 2.2s).*


**Что:** заменить regex intent classifier + per-intent assembly на **parallel fetch всех local слоёв** + **adaptive chat history compression**.

**Полная спека:** `docs/jippy_architecture_final.md` §17.8 (переписан routing tree).

**Зачем именно сейчас:** без этого S11-S14 (NER, closeness, UI) не дают полного эффекта — текущий regex-классификатор маскирует их качество. Cерия "Семих" / "Эйнштейн" / "Лайка-кличка-собаки" → разные ответы, у нас сейчас все попадают в один и тот же путь.

**Реализация:**

#### 15.1 Parallel fetch (~6 часов)

```python
# В harness.py: route() переписать
import asyncio

async def route(user_id, query, voice_target=None):
    # 1. Query rewrite (как сейчас)
    rewritten = await query_rewriter(query, chat_history)

    # 2. PARALLEL fetch всех local layers
    results = await asyncio.gather(
        layer1.search(user_id, rewritten.search_query, sender_filter=rewritten.sender_hint),
        layer2a.get_profile(user_id),                      # facts + inferences (NOT hypotheses)
        layer2b.find_people_in_query(user_id, query),      # alias lookup
        layer2c.find_concepts_in_query(user_id, query, depth=1),
        layer2e.relevant_antipatterns(user_id, query),
    )

    # 3. Conditional Layer 0
    if has_temporal_marker(query):
        layer0_result = await sonar.search(query)
    else:
        layer0_result = None  # backbone parametric handles

    return RouteResult(layers={
        "layer1": results[0],
        "layer2a": results[1],
        "layer2b": results[2],
        "layer2c": results[3],
        "layer2e": results[4],
        "layer0": layer0_result,
    })
```

**Удалить:**
- `_INTENT_PATTERNS` regex словарь
- `_classify_intent_heuristic()` функцию
- `_assemble_episodic`, `_assemble_person`, `_assemble_semantic_self`, `_assemble_world_fact`, `_assemble_generation` — все заменяются единым parallel fetch + section formatter

**Добавить:**
- `format_context_with_sections(layers_results)` — формирует prompt секции «=== PROFILE FACTS ===», «=== RELEVANT PEOPLE ===» и т.д.

#### 15.2 Adaptive context window (~3 часа)

```python
def build_messages_context(chat_history, current_msg, budget_tokens=10000):
    if count_tokens(chat_history) <= budget_tokens:
        return chat_history + [current_msg]

    last_2_turns = chat_history[-4:]
    older = chat_history[:-4]
    summary = await summarize_with_cerebras(older)

    return [
        {"role": "system", "content": f"Earlier in conversation: {summary}"},
        *last_2_turns,
        current_msg
    ]
```

Token counter — tiktoken (или прокси через символы × 0.75).

#### 15.3 Layer 0 split (~1 час)

```python
TEMPORAL_MARKERS = {
    "сегодня", "вчера", "сейчас", "новости", "курс",
    "погода", "live", "now", "current", "latest"
}

def has_temporal_marker(query):
    return any(m in query.lower() for m in TEMPORAL_MARKERS)
```

Default — **не вызываем Sonar**, экономим 1-2 сек на каждом запросе. Cerebras parametric memory покрывает established facts.

#### 15.4 Eval extension (~2 часа)

Добавить в `eval/queries.jsonl`:
- **Disambiguation tests:**
  - `"Кто такой Семих"` → expected: person from Layer 2b
  - `"Кто такой Эйнштейн"` → expected: world fact (Cerebras parametric)
  - `"Что такое Лайка"` (если есть в концептах как pet) → expected: concept from Layer 2c
- **Long-conversation test:** chat history >10K tokens → response не теряет accuracy
- **No-Sonar test:** non-temporal queries не должны вызывать Sonar (логи должны это показать)
- **Regression:** все S1-S5 queries продолжают работать

#### 15.5 Risks

- **Cerebras может «теряться»** в too much context → mitigation: явные section headers + system prompt инструкция «use most relevant section»
- **Latency:** не должна вырасти. Local parallel fetch ~100ms, Sonar убран из default → суммарно может **уменьшиться** на 1-2 сек
- **Disambiguation correctness:** покажут eval queries

**Зависит от:** ничто блокирующее. **Желательно делать после/параллельно** S11 (Layer 2c NER) — тогда concepts не мусорные и LLM лучше их использует.

---

### 14. Уборка hard_rules в harness (~30 мин)
**Что:** убрать step «hard_rules check» из `harness.py:route()`. Возвращается в Phase Б когда появится UI для управления правилами.

**Действие:**
- Удалить вызов `_check_hard_rules()` из начала `route()`
- Оставить функцию + table в schema (чтобы не делать migration сейчас)
- Добавить comment: `# hard_rules disabled until Phase Б UI lands`

**Реальные запреты** пока имитируются через system prompt инструкции в backbone calls.

---

## P2 — Phase Б deferred (специально не делал)

- iOS port (Phase B sequencing)
- BLE mesh / LoRA Hunt (§16)
- Sensor pipeline (§11)
- Tool use (§10)
- Per-relationship мини-LoRA (variant B)
- GraphRAG community detection (Leiden algorithm)
- LLM-based intent classifier (replace heuristic in `harness._classify_intent_heuristic`)
- Multi-hop knowledge graph reasoning
- Curated cache (Layer 0c)
- TTT L2/L3 (§12.3, §12.4)
- Full federated FT (Phase Г, year 2+)
- L10 personal FT ceremony (year 3+)

Эти осознанно не сделаны в MVP.

---

## Что уже работает прямо сейчас

| Endpoint | Что делает |
|---|---|
| `POST /jippy/brain/ingest {sources:["tg"]}` | Заполняет brain.db из buckets/raw_messages.jsonl |
| `GET /jippy/brain/search?q=X&k=20` | FTS5 search с фильтрами sender/chat/since/until/source |
| `GET /jippy/brain/event/:id` | Полный raw event |
| `GET /jippy/brain/timeline?sender=X` | Хронология по человеку с пагинацией |
| `GET /jippy/brain/profile` | Layer 2a профиль (Tier 1) |
| `POST /jippy/brain/profile/extract` | Перегенерить профиль |
| `GET /jippy/brain/people?limit=50` | Top-N людей с relationships |
| `GET /jippy/brain/person/:alias` | Профиль человека (любой alias) |
| `POST /jippy/brain/people/extract` | Регенерация relational graph |
| `GET /jippy/brain/concept/:name?depth=1` | Концепт + neighbors + recent events |
| `GET /jippy/brain/concepts?q=X&k=30` | Concept autocomplete |
| `POST /jippy/brain/concepts/extract` | Регенерация knowledge map |
| `GET /jippy/brain/loras` | List adapters в registry |
| `GET /jippy/brain/lora/default` | Дефолтная LoRA (jippy-v3) |
| `POST /jippy/brain/ask` | **Полный harness pipeline** (SSE streaming) |
| `GET /jippy/brain/health` | Stats — n_events, encryption, schema version |

---

## Метрики после ingest TG-истории Юки

```
events:                    608,408
n_chats:                     2,949
n_senders:                   2,861
date range:    2017-11-29 → 2026-04-27 (8.4 years)
brain.db size:                 ~1.0 GB encrypted
people in graph:                  50 (top by interaction)
concepts (after extract):     ~5,000-15,000 (depends on min_freq)
search latency:           14-340 ms (typical)
encryption:               SQLCipher AES-256
```

---

## Файлы

- `backend/migrations/001_initial.sql` — schema
- `backend/app/brain/db.py` — connection + migrations + encryption
- `backend/app/brain/ingest.py` — Layer 1 batch+realtime ingest
- `backend/app/brain/layer1.py` — search + timeline
- `backend/app/brain/extract_profile.py` — Layer 2a
- `backend/app/brain/extract_people.py` — Layer 2b
- `backend/app/brain/extract_concepts.py` — Layer 2c
- `backend/app/brain/harness.py` — Layer 4 routing (full §17.8)
- `backend/app/brain/backbone.py` — LLM client (stub/openai/mlx)
- `backend/app/brain/lora.py` — Layer 3 adapter registry
- `backend/app/routers/brain.py` — HTTP API (all endpoints above)
- `backend/eval/queries.jsonl` — 28 golden queries
- `backend/eval/run.py` — eval harness
- `backend/app/telegram/realtime.py` — Phase 0.5 — wired to mirror new TG msgs into brain.db
- `backend/app/main.py` — brain router included

---

## Запуск

```bash
cd backend
JIPPY_BACKBONE=stub uvicorn app.main:app --reload --port 8000
```

Затем `curl -H "Authorization: Bearer <jwt>" http://localhost:8000/jippy/brain/health` для смоук-теста.

Когда выберется backbone — `JIPPY_BACKBONE=openai JIPPY_BACKBONE_URL=... uvicorn ...`.
