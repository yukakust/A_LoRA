# Feature Spec: Sonar — Universal Personal Data Scanner

**Дата создания:** 2026-04-21
**Статус:** specification (реализация TBD после Jippy MVP)
**Связано с:** `jippy_architecture_final.md`, `feature_spec_import_ai_history.md`, `feature_spec_hybrid_search.md`, `project_model_marketplace.md` (Era 6)

---

## Зачем это

**Боль:** новый юзер Jippy = пустой Jippy. Нужно месяцы общения чтобы Jippy узнал тебя.

**Решение:** Sonar автоматически сканирует все источники данных юзера (комп + Google + соц сети + ...), строит **Knowledge Map** и подключает Jippy на верх. Day 1 Jippy уже знает твою историю, контекст и темы экспертизы.

**Бонус:** твоя Knowledge Map становится capability vector для Era 6 federated network — ты автоматически "expert" по своим темам в глобальной сети Jippy.

---

## Архитектура

```
[Connectors]    [Parsers]      [Normalizer]   [Indexer]     [Jippy]
Local FS    →   PDF/DOCX/MD →                 BM25 + vec →  Знает тебя
Gmail       →   Email MIME  →                 Knowledge    с Day 1
Drive       →   Docs/Sheets →   Q&A pairs →   Graph     →  + Era 6
Photos      →   VLM caption →                              federation
Browser     →   History/bookmarks
Messengers  →   Telegram/Slack/iMessage
ChatGPT/Cl. →   ZIP exports (уже в спеке)
Calendar    →   Events
Code        →   GitHub local
```

---

## Источники по приоритету

| Tier | Источники | Auth | Время интеграции |
|------|-----------|------|-------------------|
| **Tier 1 (MVP, 6 коннекторов)** | Local Documents, Browser history, Gmail, Google Drive, ChatGPT/Claude/Gemini exports, Photos | Mix | ~6 недель |
| **Tier 2** | Telegram, Slack, iMessage, Calendar, Notion, Notes (Apple/Obsidian), GitHub | Mix | +6 недель |
| **Tier 3** | Twitter/X, Facebook, Instagram, LinkedIn, WhatsApp, YouTube history, Maps history | OAuth + scrape | +6 недель |
| **Tier 4** | iCloud full, OneDrive, Dropbox, Spotify, Strava, Banking | Mix | +4 недели |

**MVP Sonar = только Tier 1 = ~3 месяца разработки.**

---

## Privacy model — критический слой

3 tier'а контроля **per source** (юзер выбирает явным toggle'ом):

| Tier | Что происходит | Default для |
|------|----------------|-------------|
| **L1: Local-only** | Данные никогда не покидают устройство. Embeddings локально. Search локально. | Email, Photos, Messengers |
| **L2: Local + encrypted backup** | Encrypted at rest на сервере, decryption key только на устройстве | Documents, Notes |
| **L3: Server-processed с явным consent** | Для тяжёлого compute (VLM captioning фоток, ASR транскрипция аудио). Опционально. | Photos VLM, Audio |

**Принцип:** raw data никогда не передаётся на сервер без explicit consent. Server видит только embeddings + synthesized hints (Era 4 Membrane Principle).

---

## Pipeline компоненты

### 1. Connector Framework
- Plugin-based архитектура (один connector на источник)
- Lifecycle: Authenticate → Initial scan → Incremental sync → Live updates
- Standard interface: `connector.fetch(since_timestamp) → list[RawItem]`
- Auth через готовые библиотеки (Nango / Pipedream / OAuth0)

### 2. Parsers (per data type)
- **Email**: MIME parse, HTML→text, attachment extraction
- **Documents**: PDF (PyMuPDF), DOCX (python-docx), MD (raw), TXT
- **Images**: VLM caption через Qwen2.5-VL-7B (уже в проде на Vadim)
- **Audio**: Whisper transcription (опционально)
- **Video**: ffmpeg frame sample + ASR
- **Code**: AST parse → structure extraction

### 3. Normalizer
- Все данные → unified Q&A pair format (`jippy_architecture_final.md` schema)
- Метаданные: source, timestamp, original_id, privacy_tier
- Дедупликация (hash-based)

### 4. Indexer
- **Hybrid Search** index (BM25 + vector) — спецификация в `feature_spec_hybrid_search.md`
- **Knowledge Graph**: entities (people, places, concepts) + relations
- Storage: PostgreSQL + pgvector (server-side L2/L3) или SQLite + sqlite-vss (local L1)

### 5. Knowledge Map UI
- Visual cluster map (auto-clustering по embeddings)
- Размер кластера = объём данных
- Связи между кластерами = co-occurrence
- Временная ось (когда о чём думал/писал)
- **Gaps highlighted** — где Jippy не имеет данных

---

## Knowledge Map — что юзер видит

```
                 [Health & Sport]
                   ●●● 47 docs
                   /    \
        [Cooking]      [Travel]
        ●●● 23         ●●●●● 89
            \           /
        [Family ●●●●●●●●● 234]
            /           \
       [Work]        [Hobbies]
       ●●●●●● 156    ●●● 34
           \           /
        [Code & Tech]
        ●●●●●●●● 312
```

---

## UX flow для юзера

1. **Onboarding:** "Хочешь чтобы Jippy сразу знал тебя? Подключи источники."
2. **Source selection:** список Tier 1 коннекторов с toggle'ами + privacy tier per source
3. **Auth flow:** OAuth для каждого выбранного источника (или upload ZIP/file picker)
4. **Initial scan:** progress bar (может занять часы для большого Gmail)
5. **Knowledge Map reveal:** показ карты, "вот что Jippy теперь знает о тебе"
6. **First chat:** Jippy уже использует контекст в первом ответе

---

## Что Jippy получает на старте

**Без Sonar (текущий MVP):**
- Cold start → ничего не знает → понимание через диалог за месяцы

**С Sonar:**
- Day 1: уже знает темы экспертизы, людей, события, паттерны
- Может: "помнишь ты обсуждал X 3 месяца назад? У меня твои заметки"
- Может: "у меня нет данных по Y, можешь рассказать?" (closes the gap proactively)
- Может: использовать твой стиль письма (из email/messages) сразу

---

## Landscape — кто уже что делает

| Продукт | Что делает | Чего не хватает |
|---------|------------|-----------------|
| **Rewind.ai** | Records all on Mac, AI search | Только Mac, не cross-source, нет AI поверх |
| **Microsoft Recall** | То же для Windows | Privacy скандал, single-OS |
| **Limitless Pendant** | Wearable + transcription | Только аудио |
| **Mem.ai** | Knowledge management | Manual, no auto-import |
| **Heptabase** | Visual knowledge graph | Manual, no AI |
| **Reflect Notes** | Notes with AI | Только notes |
| **Granola** | Meeting notes | Только встречи |

**Ниша свободна** для cross-source + personal AI поверх + federation. Это наша позиция.

---

## Технические риски и mitigation

| Риск | Mitigation |
|------|------------|
| OAuth scale (20+ источников, разные quirks) | Использовать готовые лайбы (Nango, Pipedream) для абстракции |
| Volume (100GB+ per user) | Progressive: только embeddings/index, raw on-device, download on demand |
| Live sync после initial scan | Webhooks где возможно, polling раз в час где нет |
| VLM/ASR cost (тяжёлая обработка) | Batch processing, cache, optional, server tier с consent |
| Cross-device (Mac/iPhone/Win) | Device-local agents, sync через encrypted vault |
| API rate limits (Google etc) | Token bucket, exponential backoff, resume from checkpoint |
| Format diversity (старые форматы, broken files) | Skip-with-log на ошибках, не блокировать pipeline |

---

## Ресурсы для MVP Sonar (Tier 1)

| Задача | Время | Кто |
|--------|-------|-----|
| Connector framework + 6 коннекторов (Tier 1) | 5-6 недель | 1 backend dev |
| Parsers (PDF/DOCX/email/HTML/image VLM) | 2 недели | 1 dev (parallel) |
| Normalizer → Q&A pairs | 1 неделя | reuse Jippy spec |
| Knowledge Map UI (web) | 2 недели | 1 frontend dev |
| Privacy controls + encryption | 1-2 недели | 1 dev |
| **Total** | **~3 месяца** | **1-2 разработчика + дизайнер** |

---

## Связь с существующей архитектурой

- **Расширяет** `feature_spec_import_ai_history.md`: сейчас 1 источник (ChatGPT ZIP), Sonar → 6+ источников автоматически
- **Использует** `feature_spec_hybrid_search.md`: тот же индекс, расширенный data sources
- **Использует** `jippy_architecture_final.md`: те же Q&A pairs, та же Permanent Storage
- **Кормит** Era 6 (`project_model_marketplace.md`): твоя Knowledge Map = capability vector для federated network

**Это build-on, не replace.** Спеку Jippy менять не нужно, только добавить connector layer.

---

## Roadmap позиционирование (3 опции для решения)

| Опция | Описание | Когда |
|-------|----------|-------|
| **A** | Sonar MVP как отдельный roadmap после Jippy MVP | Q3 2026 |
| **B** | Минимальный Sonar (3 коннектора) внутри Jippy MVP | +1 мес к текущему плану |
| **C** | Только спецификация сейчас, реализация после product-market fit Jippy | TBD |

**По умолчанию: C** (этот документ — спецификация, реализация позже). Решение принимается после Jippy MVP launch и feedback от первых юзеров.

---

## Open questions

1. **iOS/Android коннекторы** — ограничения Apple/Google на доступ к данным других приложений могут заблокировать часть Tier 1 (iMessage, Photos full access). Решение: web-based imports + Share Sheet интеграция.
2. **Encryption keys** — где хранить мастер-ключ юзера для L2 encrypted backup? Варианты: device-only (теряется при потере устройства), iCloud Keychain, recovery phrase.
3. **Knowledge Graph schema** — какие entity types и relations? Базовая онтология vs auto-discovered.
4. **Multi-device sync** — несколько устройств юзера видят одну Knowledge Map или каждое свою?
5. **Update cadence** — initial scan один раз vs continuous re-scan для апдейтов.
