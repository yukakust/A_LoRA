# Jippy — "Свой ИИ" — Полная архитектура системы

**Версия:** 2.0  
**Дата:** 2026-04-15  
**Статус:** Утверждено  

---

## 1. Продукт

**Hero:** "Свой ИИ."  
**Lead:** "Научи один раз — знает навсегда. Не объясняй с нуля. Не делись ни с кем."

**Боль:** У всех один и тот же ChatGPT. Он не помнит тебя. Ты объясняешь одно и то же с нуля. Твой ИИ ничем не отличается от чужого.

**Решение:** ИИ который ТВОЙ. Живёт на твоём телефоне. Обучен на ТВОИХ данных. Помнит всё. Не делится ни с кем. И при этом работает как топовый ИИ даже на незнакомых темах (серверный backbone).

---

## 2. Принципы

1. **Храни ВСЁ.** Все данные пользователя хранятся ВСЕГДА. Ничего не удаляется, не суммаризуется, не сжимается. Raw verbatim. (Подтверждено: MemPalace доказал что raw > summary, 96.6% vs 84.2% retrieval accuracy.)
2. **Q&A пара = атомарная единица.** Каждая пара вопрос+ответ — единица хранения, тренировки и поиска.
3. **Jippy всегда полезен.** Даже на незнакомых темах Jippy даёт user context (кто юзер, его стиль, предпочтения). На знакомых темах — добавляет domain-specific hints.
4. **Повторение — мать учения.** Каждый level-up = полный ретрейн с нуля на ВСЕХ данных, не инкрементальный.

---

## 3. Компоненты системы

### 3.1 Телефон

#### Qwen3.5-0.8B — "Jippy" (модель на устройстве)
- **Размер:** ~500MB в INT4
- **Мультимодальная:** текст + картинки + видео
- **Роль:** генерирует hint для серверного backbone
- **Два выхода:**
  - User Profile context (ВСЕГДА): кто юзер, предпочтения, стиль
  - Domain Hint (только на знакомых темах): конкретные знания по вопросу
- **Обновляется:** при каждом level-up (новая модель скачивается с сервера)

#### Permanent Storage (хранилище на устройстве)
- ВСЕ Q&A пары, raw verbatim
- User profile
- Embedding-индекс для similarity search
- Синхронизация с сервером
- Формат: SQLite + FTS5 (keyword search) + embedding index

#### RAG Buffer (временный)
- Только НОВЫЕ знания после последнего level-up
- Используется для дополнения модели данными которые она ещё не "выучила"
- Сбрасывается после level-up (данные остаются в storage)

### 3.2 Сервер

#### Backbone Model (топовая open-source модель)
- Одна и та же для ВСЕХ пользователей
- Получает hint от Jippy в system prompt
- Генерирует финальный ответ
- Текущий выбор: лучшая доступная мультимодальная модель

#### Training Pipeline (GPU)
- Полный LoRA fine-tune при level-up
- Base Qwen3.5-0.8B + ВСЕ данные пользователя → новая модель
- Время: ~10-40 мин на H100 (до 45K данных)
- Результат: INT4-квантизованная модель → деплой на телефон (~500MB)

#### Hybrid Search Engine
- Поиск по ПОЛНОМУ хранилищу пользователя
- BM25 (keyword exact match) + Vector (semantic similarity)
- Используется для: confidence gating (инджектить или нет) + RAG retrieval
- Реализация: PostgreSQL FTS + embedding search (или ChromaDB)

#### Import Pipeline
- Парсинг ZIP-файлов (ChatGPT, Claude, Gemini)
- Извлечение Q&A пар + User Profile
- Генерация embeddings
- Light mode: text → profile extraction (<5 сек)
- Full mode: ZIP → parse → extract → embed (<2 мин)

---

## 4. Потоки данных

### 4.1 Inference Flow (каждый вопрос)

```
ПОЛЬЗОВАТЕЛЬ ЗАДАЁТ ВОПРОС
         │
         ▼
┌────────────────────────────┐
│  STEP 1: PERPLEXITY CHECK  │
│  Qwen3.5-0.8B считает     │
│  perplexity на вопросе     │
└─────────┬──────────────────┘
          │
          ├── Низкая perplexity ──► ЗНАКОМАЯ ТЕМА
          │                         │
          │                         ▼
          │               Qwen3.5-0.8B генерирует:
          │               • User Profile (кто юзер)
          │               • Domain Hint (знания по теме)
          │               • RAG buffer дополняет (если есть новое)
          │                         │
          │                         ▼
          │               System prompt для backbone:
          │               ┌──────────────────────────┐
          │               │ USER CONTEXT: {profile}  │
          │               │ EXPERT KNOWLEDGE: {hint}  │
          │               │ ADDITIONAL: {RAG data}    │
          │               │                          │
          │               │ Answer using the expert  │
          │               │ knowledge as primary.    │
          │               └──────────────────────────┘
          │
          └── Высокая perplexity ──► НЕЗНАКОМАЯ ТЕМА
                                     │
                                     ▼
                           Qwen3.5-0.8B генерирует:
                           • User Profile ТОЛЬКО
                           • БЕЗ domain hint (был бы мусор)
                                     │
                                     ▼
                           System prompt для backbone:
                           ┌──────────────────────────┐
                           │ USER CONTEXT: {profile}  │
                           │                          │
                           │ Answer based on your own │
                           │ knowledge. Adapt style   │
                           │ to user context.         │
                           └──────────────────────────┘

                  ▼ (в обоих случаях)
┌─────────────────────────────────────┐
│  STEP 2: BACKBONE ОТВЕЧАЕТ          │
│  Топовая open-source модель         │
│  получает system prompt + вопрос    │
│  → генерирует ответ                 │
└─────────┬───────────────────────────┘
          │
          ▼
┌─────────────────────────────────────┐
│  STEP 3: СОХРАНЕНИЕ                 │
│  Q&A пара → permanent storage       │
│  Новое → RAG buffer                 │
│  (следующий раз тема более знакома) │
└─────────────────────────────────────┘
```

### 4.2 Level-Up Flow (при накоплении данных)

```
ДАННЫХ ДОСТАТОЧНО ДЛЯ LEVEL-UP
         │
         ▼
┌────────────────────────────┐
│  STEP 1: UPLOAD            │
│  Все данные из storage →   │
│  сервер (зашифрованные,    │
│  временно, только для      │
│  тренировки)               │
└─────────┬──────────────────┘
          │
          ▼
┌────────────────────────────┐
│  STEP 2: FULL RETRAIN      │
│  Base Qwen3.5-0.8B         │
│  + ВСЕ данные пользователя │
│  → LoRA fine-tune           │
│                            │
│  Время: ~10-40 мин (H100)  │
│  Фоновая задача            │
└─────────┬──────────────────┘
          │
          ▼
┌────────────────────────────┐
│  STEP 3: DEPLOY            │
│  LoRA merge → INT4 quant   │
│  → ~500MB модель           │
│  → скачивается на телефон  │
│  → замена старой модели    │
└─────────┬──────────────────┘
          │
          ▼
┌────────────────────────────┐
│  STEP 4: CLEANUP           │
│  RAG buffer сбрасывается   │
│  (знания теперь в весах)   │
│  Тренировочные данные      │
│  удаляются с сервера       │
│  Storage на телефоне — без  │
│  изменений                 │
└────────────────────────────┘
```

### 4.3 Import Flow (cold start)

```
НОВЫЙ ПОЛЬЗОВАТЕЛЬ
         │
         ├── Light Mode (30 сек)
         │   │
         │   ▼
         │   Копирует текст из ChatGPT/Claude memory
         │   → POST /api/jippy/import/text
         │   → LLM extraction → User Profile
         │   → Storage + RAG
         │   → Jippy сразу персонализирован
         │
         └── Full Mode (5-10 мин)
             │
             ▼
             Загружает ZIP (ChatGPT/Claude/Gemini)
             → POST /api/jippy/import/upload
             → Parse conversations → Q&A pairs
             → Extract User Profile
             → Generate embeddings
             → Storage + RAG
             → Jippy знает всю историю
```

---

## 5. Хранение данных

### Единица хранения: Q&A пара

```json
{
  "id": "sha256(question+answer+timestamp)",
  "question": "нормализованный вопрос для поиска",
  "answer": "нормализованный ответ для поиска",
  "raw_question": "оригинальный вопрос verbatim",
  "raw_answer": "оригинальный ответ verbatim",
  "timestamp": "2025-06-15T14:30:00Z",
  "source": "chat | import_chatgpt | import_claude | import_gemini | feed",
  "topic_keywords": ["analytics", "google"],
  "embedding": [float array]
}
```

### Принцип: Store Raw, Search Smart

- **Хранение:** raw verbatim, без суммаризации. Оригинал НИКОГДА не изменяется и не удаляется.
- **Поиск:** Hybrid (BM25 keyword + vector semantic). Находит по точным словам И по смыслу.
- **Тренировка:** normalized Q&A пары (без мусора, но полный контент).
- **Дедупликация:** по hash(question + answer + timestamp). Повторный импорт не создаёт дубликатов.

### Три слоя доступа к данным

| Слой | Что содержит | Где | Lifecycle |
|------|-------------|-----|-----------|
| **Permanent Storage** | ВСЕ Q&A пары + profile, raw | Телефон + server sync | Вечный, только растёт |
| **RAG Buffer** | Новое после последнего level-up | Телефон (+ server) | Сбрасывается при level-up |
| **Model Weights** | Знания в весах Qwen3.5-0.8B | Телефон (~500MB) | Перезаписывается при level-up |

---

## 6. Level-Up: пороги данных

| Level | Кумулятивно Q&A пар | Delta |
|-------|---------------------|-------|
| 1 | 5,000 | — |
| 2 | 11,000 | +6,000 |
| 3 | 19,000 | +8,000 |
| 4 | 30,000 | +11,000 |
| 5 | 45,000 | +15,000 |
| 6 | 65,000 | +20,000 |
| 7 | 91,000 | +26,000 |
| 8 | 124,000 | +33,000 |
| 9 | 165,000 | +41,000 |
| 10 | 215,000 | +50,000 |

Формула delta: +6K, +8K, +11K, +15K, +20K, +26K, +33K, +41K, +50K

---

## 7. Приватность

| Данные | Где хранятся | Когда уходят на сервер |
|--------|-------------|----------------------|
| Q&A пары | Телефон (primary) | Level-up (ретрейн) — временно, удаляются после |
| User Profile | Телефон | Level-up — часть тренировочных данных |
| Модель Qwen3.5-0.8B | Телефон | Никогда — скачивается С сервера |
| Вопрос юзера | Телефон | При запросе к backbone — hint + вопрос |
| Hint от Jippy | Генерируется на телефоне | Отправляется backbone вместе с вопросом |
| ZIP-импорт | Загружается на сервер | Удаляется после парсинга |

**Ключевое:** на сервер уходит минимум — hint (короткий текст) + вопрос. Не вся история. Модель и данные живут НА телефоне.

---

## 8. Серверные API endpoints

### Import
```
POST   /api/jippy/import/text          — Light mode (text paste)
POST   /api/jippy/import/upload        — Full mode (ZIP upload)
GET    /api/jippy/import/status/:id    — Progress polling
GET    /api/jippy/import/history       — Previous imports
```

### Inference
```
POST   /api/jippy/ask                  — Main endpoint:
         Body: { question, hint, user_context, mode: "familiar"|"unknown" }
         Server: injects into backbone system prompt → returns answer
```

### Search (hybrid BM25 + vector)
```
POST   /api/jippy/search               — Search user's storage
         Body: { query, limit, threshold }
         Returns: relevant Q&A pairs ranked by hybrid score
```

### Level-Up
```
POST   /api/jippy/levelup/start        — Upload data, start training
GET    /api/jippy/levelup/status/:id   — Training progress
GET    /api/jippy/levelup/download/:id — Download new model (INT4)
```

### State
```
GET    /api/jippy/state                 — Current level, data count, profile
POST   /api/jippy/feed                 — Add Q&A pair manually
GET    /api/jippy/knowledge            — List stored knowledge
```

---

## 9. Зависимости между командами

### Mobile App Team:
- Document/File Picker для ZIP
- Share Extension (iOS) / Share Target (Android)
- Qwen3.5-0.8B inference on-device (ONNX Runtime / llama.cpp)
- Local SQLite + FTS5 для storage + RAG
- Multipart upload для ZIP
- Model download + replacement при level-up

### Server Team:
- ZIP parsing pipeline (ChatGPT/Claude/Gemini formats)
- Profile extraction (LLM-powered)
- Hybrid search engine (BM25 + vector)
- LoRA training pipeline (Qwen3.5-0.8B)
- INT4 quantization + model serving
- Backbone model hosting (inference)

### Shared:
- API contract (endpoints above)
- Q&A pair format (normalized JSON)
- Auth + encryption for data sync
- Embedding model alignment (same model on server and phone for consistent search)

---

## 10. Tool Use на телефоне

**Тезис:** телефон для tool use **лучше облака**. У него уже есть инструменты, и доступ к контексту юзера которого никогда не будет у ChatGPT.

### 10.1 Что Jippy умеет звать как инструмент

| Инструмент | Что делает | Платформа |
|---|---|---|
| Калькулятор / math runtime | Точные вычисления | iOS/Android встроено |
| Календарь | Читать события, создавать, искать | EventKit (iOS) / CalendarContract (Android) |
| Контакты | Искать, читать поля | Contacts API |
| Заметки / Reminders | Читать существующие, создавать новые | Notes/Reminders intents |
| Браузер | Открыть URL, прочитать страницу | WKWebView / WebView |
| Локальный JS/Python sandbox | Выполнить код | Pyodide / JavaScriptCore |
| Любое приложение через Intents | Открыть Spotify на песне, отправить в Messenger, etc. | iOS App Intents / Android Intents |
| **Accessibility API** | **Видеть что у юзера на экране в любом приложении** | UIAccessibility / AccessibilityService |
| Camera / Photos | Снять, прочитать галерею | Photos framework |
| Микрофон + ASR | Принять голосовой ввод | Speech framework / SpeechRecognizer |
| Clipboard | Прочитать что скопировано | UIPasteboard / ClipboardManager |

### 10.2 Архитектура tool calling

```
ВОПРОС ЮЗЕРА → Jippy (Qwen3.5-0.8B)
                  │
                  ▼
        Решает: нужен tool или нет?
                  │
        ┌─────────┼─────────┐
        ▼         ▼         ▼
     Calendar  Browser  Accessibility
        │         │         │
        └─────────┼─────────┘
                  ▼
       Результат tool возвращается
       в контекст Jippy → финальный ответ
```

### 10.3 Уникальное преимущество vs облако

ChatGPT не видит:
- Что у тебя сейчас открыто в Telegram
- Что в твоём календаре через 2 часа
- Кому ты звонил утром
- Что ты скопировал минуту назад
- Что на экране в твоём банковском приложении

Jippy через Accessibility + системные API **видит всё это** (с разрешения юзера, локально, без выгрузки). Это **структурно невозможно** для облачной модели.

---

## 11. Sensor Pipeline — что захватываем и зачем

**Принцип:** **не снимаем видео**. Не пишем микрофон постоянно. Это убивает батарею и приватность. Захватываем точечно — событийно или по запросу.

### 11.1 10 практических сценариев захвата

1. **Скриншот → анализ.** Юзер скринит чат → Jippy парсит OCR → запоминает с кем общался, о чём, какой стиль. Энергозатрат — секунды OCR на фрейм.

2. **Accessibility snapshot.** При запросе «помоги с этим экраном» Jippy читает текущий экран приложения через accessibility API. Без камеры. Без записи. Только когда юзер просит.

3. **OCR с камеры по запросу.** Юзер фотографирует страницу книги / меню / документа / упаковки → текст в storage с тегом темы. Один кадр, не видео.

4. **ASR на голосовых заметках.** Юзер надиктовал мысль → speech-to-text → Q&A пара в storage с метаданными (время, локация). Не постоянная запись — только когда юзер нажал кнопку.

5. **Календарь + контакты.** Jippy знает что у тебя встреча с Игорем через час. Может предложить «хочешь подготовлю контекст?» Подтягивает прошлые разговоры с Игорем из storage.

6. **App usage паттерны.** Какие приложения юзер использует много (Cursor / Figma / chess.com → программист, дизайнер, шахматист). Влияет на стиль ответов и domain hints. Без слежки за контентом — только метаданные использования.

7. **Clipboard с фильтром.** Юзер скопировал что-то → если выглядит как ценная инфа (URL, цитата, имя, длинный текст) — спросить «сохранить в Jippy?». То что юзер копирует — он считает важным.

8. **Локация + время как контекст.** Паттерны жизни: где работаешь, где живёшь, маршруты. Для ответов («ты сейчас в офисе, отвечаю короче») и фоновых рекомендаций.

9. **Голосовая команда без открытия приложения.** Hot-key или Siri/Assistant intent → короткая мысль → сразу в storage. Снижает трение «надо открыть приложение → набрать».

10. **Биометрия с Apple Watch / Wear OS.** Сердечный ритм, сон, активность. Контекст состояния («ты не выспался, делаю короткий ответ»). Опционально, через Health/Fit API.

### 11.2 Что НЕ делаем (и почему)

| Не делаем | Причина |
|---|---|
| Постоянная запись видео | Батарея + хранение убиваются за день |
| Постоянная запись микрофона | Батарея + приватность + регуляции |
| Скрытое фотографирование | Доверие |
| Захват без явного разрешения | Доверие, App Store отказ |
| Выгрузка raw сенсорных данных в облако | Ломает «свой ИИ» обещание |

### 11.3 Pipeline

```
Сенсор/событие → Privacy filter → Importance filter
                                        │
                                        ▼
                      Encode (CLIP / text embedding)
                                        │
                                        ▼
                          Local vector DB (SQLite + FAISS)
                                        │
                                        ▼
                ┌───────────────────────┴───────────────────────┐
                ▼                                                ▼
        RAG retrieval                              Periodic LoRA fine-tune
        (в момент запроса)                         (раз в N дней)
```

**Importance filter** критичен — иначе 10 ГБ за неделю. Большинство дня — мусор. Сохраняем только то что: повторяется, явно отмечено юзером, попадает в его темы, или содержит конкретные факты (имена, числа, даты).

---

## 12. Continuous Learning — три уровня TTT

**Тезис:** *модель учится сейчас на твоей конкретной проблеме*. Облачные не делают и не сделают. Это наш ⭐⭐⭐ козырь.

### 12.1 Лестница реализации

| Уровень | Что | Сложность | Срок |
|---|---|---|---|
| **L1: Correction Cache** | Юзер поправил Jippy → пара (мой_ответ, правильный) сохраняется с высоким приоритетом в retrieval | За день | Делаем сразу |
| **L2: Session-LoRA** | Микро-LoRA на текущий разговор. В конце сессии: отбросить или смерджить в базовую если ценно | За неделю | Phase Б |
| **L3: Real on-device TTT** | Backprop на телефоне, обновление весов в реальном времени | Сложно (фреймворки сырые) | 12+ месяцев |

### 12.2 L1 — Correction Cache (делать первым)

**Логика:**
1. Юзер задаёт вопрос → Jippy отвечает
2. Юзер поправляет: «нет, на самом деле X»
3. Сохраняем в storage: `{question, jippy_answer, user_correction, timestamp, priority: HIGH}`
4. При следующем похожем вопросе — retrieval **вытаскивает correction первой строкой** и кладёт в контекст: «прошлый раз ты ошибся вот так, юзер поправил вот так, не повторяй»
5. Через N таких поправок по похожим темам — попадают в обязательный fine-tune

**Не настоящее обучение весов**, но эффект функционально похожий и ощущается как «он сразу учится». Юзер не отличает.

### 12.3 L2 — Session-LoRA

**Логика:**
1. Старт сессии → инициализируем пустую микро-LoRA (rank 4, ~5MB)
2. В течение сессии — собираем правки и явные «запомни это»
3. В конце сессии — короткий fine-tune (минуты, batch=небольшой) на собранном
4. Если ценно (>N полезных пар) — мерджим в базовую LoRA. Иначе — отбрасываем.

**Требует:** локальный fine-tuning стек на телефоне (Apple Neural Engine + MLX или ONNX Runtime + custom backprop). Уже частично возможно на новых iPhone.

### 12.4 L3 — Real TTT (research-этап)

Backprop на устройстве в момент использования. Apple MLX framework движется в эту сторону. Edge Impulse, CoreML — частично умеют. **На 1-2 года вперёд.** Не строим сейчас, но архитектуру оставляем совместимой.

### 12.5 Маркетинговое обещание

> «Поправь один раз — Jippy запомнит навсегда.»

Это правда уже на L1. На L2 правда становится буквальной (веса обновляются). На L3 — мгновенной.

### 12.6 Conservative weight updates от внешних источников

**Тезис:** *знакомые могут менять твои убеждения, незнакомцы — только информировать*. Один принцип закрывает 90% manipulation surface.

Когда LoRA получает knowledge card извне (другой юзер, mesh, LoRA Hunt — см. §16), она НЕ применяется к весам сразу. Card падает в RAG-буфер с провенансом и ждёт триггера промоута.

**Источники provenance:**
- **(a) Личный опыт ошибки.** Юзер сам обжёгся, поправил Jippy — это L1 correction. Золотой стандарт.
- **(b) Сигнал от close circle.** Только из явного whitelist (20-50 человек, добавлены юзером явно), AND grounded в их собственном (a), AND k≥3 таких в whitelist принесли тот же сигнал.
- **(c) Manual «выучи это».** Только сам себе можешь. Никто другой через (c) не влияет.

**Незнакомцы** — их cards ходят в RAG-буфере для retrieval (могут информировать ответ), но **никогда** не промоутятся в веса. В жизни ты тоже не меняешь убеждения от слов прохожего на улице.

**Decay по хопам:**
- (a)-source = вес 1.0
- (b) на 1 хоп от (a) = 0.7
- (b) на 2 хопа = 0.4
- (b) на 3+ хопов = 0.1

Длинные цепочки сплетен теряют силу — как в реальной жизни.

**Card structure:**
```json
{
  "topic": "грузинские квеври",
  "insight": "...",
  "provenance": [
    {"source": "self", "how": "(a)error_correction",
     "evidence": "ошибся 2026-04-15 в задаче X"}
  ],
  "signed_by": "<ed25519_pubkey>",
  "timestamp": "2026-04-27T14:00:00Z"
}
```

**Quality control — outcome-based only (RLVR-style).**

Никаких human votes (легко фармятся). Один объективный сигнал:

```
Когда card используется (retrieved + влияет на ответ):
  если юзер не исправил, не пожаловался, задача решена → utility += 1
  если юзер исправил / задача провалена → utility -= 1
```

Через N использований у каждой card накоплен **objective utility score** — фактический сигнал того, помогает она или мешает. Это и есть state-of-art подход в современном RL (RLVR / RLEF — reward from verifiable execution feedback). Cards с низким utility понижаются в весе или удаляются автоматически.

**Промоут в веса:**
- Из RAG-буфера в обязательный fine-tune попадает card если: provenance валиден (см. правила выше) AND utility_score ≥ threshold AND card не противоречит другим high-utility cards
- Раз в N дней — батч-апдейт LoRA из промоутнутого пула

---

## 13. Form Factors — surfaces продукта

Jippy — **один продукт на нескольких surfaces**. Все они делят: данные юзера, идентичность, тренировочный pipeline, аккаунт, маркетинг. Различаются только UX и доступными сенсорами.

### 13.1 Mobile (primary surface)

**Статус:** primary, описан в разделах 1-12.

**Уникальное:** камера, микрофон, accessibility API, локация, биометрия (Apple Watch), всегда с юзером.

**Платформы:** iOS (Swift + Core ML / MLX), Android (Kotlin + ONNX Runtime).

**Модель:** Qwen3.5-0.8B INT4, ~500MB.

### 13.2 Chrome Extension

**Статус:** активен (см. memory `project_clip_infrastructure.md` — Map tab с 1-click crawl 600 уже работает).

**Роль:** захват веб-контекста, на чём юзер сидит в браузере. Источник тренировочных данных.

**Что делает:**
- Видит что юзер читает / смотрит / посещает (с разрешения)
- One-click сохранение в Jippy storage («запомни эту страницу»)
- Inline помощь на странице (как Cursor для веба)
- Crawl-режим — бэтч-сбор тематического контента (research mode)

**Что НЕ делает:** тренирует модель локально (это backend task).

**Sync:** пишет в общий Jippy storage юзера (через API), читается мобильной моделью.

### 13.3 Desktop App

**Статус:** планируется (см. memory `project_implementation_plan.md` — порядок: API → Chrome Ext → App MVP → Desktop → Full App).

**Роль:** для тех кто работает с компьютера. Тяжёлые задачи (большой контекст, code, документы).

**Что делает:**
- Запуск более крупной модели локально (например Qwen3.5-7B вместо 0.8B), если железо позволяет
- Доступ к файлам / документам / коду на машине
- Глобальный hot-key для голосового/текстового запроса
- Screen capture с разрешения (как мобильный accessibility, но для десктопа)

**Платформы:** macOS (приоритет, MLX-нативный), затем Windows/Linux.

**Sync:** читает/пишет в общий Jippy storage.

### 13.4 Public API

**Статус:** частично live (см. memory `project_deployment_status.md` — extract-json live, ожидают: summarize, tech-ru2en, FLUX.2 image gen).

**Роль:** программный доступ к Jippy для разработчиков и интеграций.

**Что отдаёт:**
- Inference endpoint (мой Jippy отвечает на вопрос)
- Specific specialist endpoints (extract-json, summarize, etc.)
- Batch endpoints для тяжёлых задач

**Хост:** gpu.social/jippy/*, Vadim Ubuntu backend + Hetzner reverse tunnel + Caddy.

**Auth:** API key per user; rate limit; usage metering для будущей монетизации.

### 13.5 Roadmap surfaces

Не строим сейчас, но архитектура совместима:
- **Voice-only device** (типа AirPods + minimal phone interaction)
- **Watch app** (Apple Watch / Wear OS) — короткие запросы, биометрия
- **CarPlay / Android Auto** — голос за рулём
- **Smart home** — встраивание в скриптовые сценарии

---

## 14. Sync между surfaces

### 14.1 Single Source of Truth

**Storage юзера:** primary копия на телефоне, реплика на сервере (зашифрованная). Сервер — для sync, не для primary.

**Все surfaces читают/пишут через сервер**, потому что прямой phone-to-extension peer sync ненадёжен.

```
Mobile (primary) ◄──► Server (sync hub) ◄──► Chrome Ext
                            ▲
                            │
                            ▼
                       Desktop App
```

### 14.2 Что синхронизируется

| Сущность | Где primary | Реплика |
|---|---|---|
| Q&A пары (storage) | Mobile | Server (зашифровано), Desktop, Chrome Ext (только читают) |
| User Profile | Mobile | Server, все остальные (read-only) |
| LoRA веса | Server (training output) | Mobile (full), Desktop (если есть железо), Chrome/API (через server) |
| RAG buffer | Mobile | Server, остальные читают |
| Settings / preferences | Server | Все читают |

### 14.3 Конфликты

При одновременной правке с разных surfaces — last-write-wins по timestamp. Q&A пары неизменяемые (raw verbatim), так что конфликт возможен только в profile / settings, где он редкий и низкорисковый.

### 14.4 Offline mode

- **Mobile** — работает полностью offline (LoRA + storage на устройстве)
- **Chrome Ext** — требует сеть (sync через server)
- **Desktop** — может работать offline (если модель скачана), но для sync нужна сеть
- **API** — server-side, всегда online

**Правило:** primary surface (mobile) должен работать без сети. Остальные могут зависеть.

---

## 15. On-Device Training (validated 2026-04-27)

**Что валидировано:** полный цикл "TG history → on-device LoRA training → working voice model" — БЕЗ серверной GPU. На M2 Mac у Юки.

### 15.1 Pipeline

```
TG backfill (личные + admin) → 691K msgs / 3.4K dialogs
  ↓ Phase D realtime listener (новые msgs auto-flow)
SFT bucket: reply_pairs.jsonl (142K pairs)
  ↓ prep_dataset.py: filter close_circle, target ≥ 3 chars
  ↓ MLX-LM messages format, mask_prompt=true
train.jsonl 19K + valid.jsonl 1K
  ↓ mlx_lm.lora --config jippy-v1.yaml
adapters/jippy-v1/0000200_adapters.safetensors (14.5 MB)
  ↓ apply_chat_template + base+adapter inference
Yuka voice on device
```

### 15.2 Hyperparams (валидированные на Qwen3.5-0.8B-4bit)

```yaml
fine_tune_type: lora
num_layers: 16
lora: { rank: 8, scale: 4.0, dropout: 0.05 }   # scale=20 ⇒ divergence
learning_rate: 5.0e-5                           # cosine + warmup 100
optimizer: adamw
batch_size: 1
grad_accumulation_steps: 4
mask_prompt: true
max_seq_length: 1024
```

Trainable params: 0.48% (3.6M из 752M). Adapter file: 14.5 MB.

### 15.3 Зеркало Phase 0 lesson: generation > loss

Третий раз подтверждаем (Phase 0, FLUX matchbox, теперь Jippy v1):

| Iter | Val loss | Generation |
|------|----------|------------|
| 200  | 4.135    | ✅ Короткие, разнообразные, релевантные ответы |
| 400  | 3.797    | ❌ Mode collapse: почти все ответы = `)))` |

iter 400 имеет **ниже val loss** потому что выучил предсказывать `)` (самый частый Yuka-токен). Lower CE, broken model.

**Правило для on-device training:** save_every малое (200 iters), eval должна включать generation samples, не только loss. Best ≠ last.

### 15.4 Crash recovery

Metal Internal Error на M2 повторился дважды после ~6h training (hardware/driver, не наш код). Mitigation: save_every=200 + `--resume-adapter-file`. Без чекпоинтов — потеря.

### 15.5 Что это означает для архитектуры

1. **Server training не обязателен** для personal voice LoRA. На phone-grade GPU (M2 — на iPhone 17 Pro будет похожее) обучение реально.
2. **Phase H "on-device delta training"** в roadmap теперь не теория — есть рабочий end-to-end на Mac. Перенос на iOS = MLX → CoreML/MPS bridge, плюс cgi-вешать save/resume.
3. **TTT (раздел 12)** становится дешевле: не серверный fine-tune, а локальный LoRA-update раз в N дней.
4. **Privacy promise усиливается**: "ваши данные никогда не покидают устройство" — теперь не только inference, но и **training**.

### 15.6 Файлы

```
training/
  prep_dataset.py            # reply_pairs.jsonl → MLX format
  configs/jippy-v1*.yaml     # hyperparams
  data/{train,valid}.jsonl   # 19K + 1K (capped from 116K close_circle)
  adapters/jippy-v1/
    adapters.safetensors     # = 0000200 (production winner)
    0000{200,400,600}_adapters.safetensors  # checkpoints
  eval_inference.py          # base vs base+LoRA on valid samples
```

### 15.7 Next iteration (jippy-v2)

Чтобы выйти за voice mimicry → semantic relevance:
- Filter target ≥ 15 chars (drops "лол", `)))`, "ага")
- Lower LR (2e-5), lower scale (2), higher dropout (0.1)
- Stop at iter 200-400 max
- Возможно rank=16 + меньше iters (capacity + regularization)

---

## 16. Proximity Mesh & LoRA Hunt

**Тезис:** AI с физическим присутствием — структурно невозможно для облака. Jippy на телефоне может **встретиться с другим Jippy в реальности** и обменяться знаниями. Это закрывает дыру #3 (centralization) и #5 (grounding) в `FRONTIER_ATTACK.md`.

### 16.1 Концепт

**Карта мира с носителями знаний. Иди → встреть → обменяйся → прокачайся.**

Не cloud federated learning, а **opt-in mesh из физических встреч**. Принципиально:
- Никакого центрального сервера для обмена
- Обмен только при физическом контакте (BLE handshake)
- Ничей LoRA напрямую не апдейтится — только knowledge cards в RAG (см. §12.6)
- Identity = коллекция тегов и уровней (Pokemon-style), не имя/лицо

### 16.2 BLE Protocol AK-47 v2

Принципы AK-47: минимум деталей, работает в грязи, нельзя сломать.

```
1. Beacon (broadcast):
   - Rotating short-term pubkey (новый каждые ~15 мин, signed by master-pubkey)
   - Topic tags юзер согласился показать публично
   - Уровень знания по каждому тегу (1-99)
   - Adaptive interval: 30s в движении, 2-5min stationary (battery)
   - НЕ показывает: имя, точные координаты, master-pubkey, контент LoRA

2. Discovery:
   - Phone видит beacons в ~10m
   - UI: "рядом 3 человека, у одного редкий тег [квеври] level 87"

3. Handshake (требуется обоюдный Accept):
   - Оба тапают "Accept" — физическое согласие, anti-spam
   - Включается nonce + timestamp в подписи (anti-replay)
   - Открывается 60-секундное окно обмена (time-window вместо count-limit)

4. Exchange:
   - Signed knowledge cards (1-10 KB каждая)
   - Подписи Ed25519 через master-pubkey
   - Card freshness: каждая card датируется, со временем вес сигнала падает
   - Incoming-only mode: можно принимать не делясь (toggle)

5. Receive:
   - Cards падают в RAG-буфер (см. §12.6) с пометкой
     "from <master-pubkey>, GPS H3 hex level 8 (~1km), timestamp, in-person"
   - НЕ применяются к LoRA — ждут utility outcome (см. §12.6)

6. Royalty event:
   - Обе стороны получают micro-payment в contribution economy
   - Автору card капает royalty за каждое использование downstream
```

**Privacy by design:**
- Точные координаты не светят (только H3 hex ~1km)
- Pubkey ротируется (anti-tracking) — стационарный наблюдатель не может построить профиль перемещений
- Бикон молчит когда global share-OFF включён

### 16.3 Failure modes которые закрыты

| Атака | Митигация |
|---|---|
| Спам близкими бикон-флудом | Лимит 60-сек окно после mutual Accept |
| Tracking через persistent pubkey | Rotating short-term keys (Apple Find My подход) |
| Replay handshake в другом месте | Nonce + timestamp в подписи |
| Кража знания пассивным sniff | Mutual Accept обязателен, exchange шифрован |
| Botferma пустых аккаунтов | Reputation accrues to master-pubkey, новый ключ = 0 trust |
| Сталкер у дома | Global share-OFF toggle — бикон молчит, всё |
| Покупка инфлюенсера для манипуляции мнением | (c) manual learn НЕ распространяется через mesh (см. §12.6) |
| Ботогенерированные cards (LLM slop) | Slop detection через perplexity profile + outcome utility отсеет автоматически |

### 16.4 LoRA Hunt — фича на этом протоколе

**5 слоёв:**

**Слой 1: Map.** Глобальная карта на H3-hex grid. Видно плотность носителей по тегам. Анонимность: видно тег + hex, не личность.

**Слой 2: Quest.** Jippy генерит квесты: «найди носителя [тег X] в радиусе Y, прокачай свою LoRA по этой теме». Reward: knowledge cards + XP + достижение в коллекции.

**Слой 3: Meet.** BLE handshake (см. §16.2). 5-минутный live exchange (живой разговор → ASR → topic extraction → knowledge cards). Опционально: «безмолвный режим» — чисто card swap без разговора.

**Слой 4: Collect.** Pokedex-стиль: коллекция тегов которые ты собрал. Редкие = бейджи, обычные = базовая прокачка. Leaderboard по регионам и темам.

**Слой 5: Host.** Юзер объявляет себя host: «я в кафе X в 18:00, у меня тег [квеври], приходите». Эвенты, конфы, тематические встречи. Туризм по уникальным локальным тегам.

### 16.5 Identity-as-mastery (Pokemon-style)

Внешняя identity не имя, не лицо, а **коллекция тегов с уровнями + бейджи**:

```
@user_42a1
🍷 wine (georgian quevri) — 73
⚛️ react — 45
📐 differential geometry — 12
[badges: Tbilisi-traveler, 100-meets, rare-collector]
```

Это **identity-as-mastery** вместо identity-as-self. Что ты знаешь и чем готов делиться — это и есть твой публичный профиль. Имя/лицо опциональны и приватны.

### 16.6 Гранулярный opt-in

Каждый тег имеет **свой on/off + level**:

```
[✓] grain wine          level 73
[✓] grain react         level 45
[ ] grain psychotherapy level 68    ← скрыт
[ ] grain personal_life level 12    ← скрыт
```

**Дефолт: все теги OFF.** Юзер явно включает каждый тег который готов показывать. Privacy-first.

**Глобальный switch share-OFF:** один тогл — бикон молчит, никто не видит. Всё. Простота AK-47.

### 16.7 Quality control — outcome-based only

См. §12.6. Никаких human-votes (фармятся). Только objective utility:
```
card используется → задача решена правильно → utility += 1
card используется → задача провалена / юзер исправил → utility -= 1
```
Cards с накопленным низким utility понижаются в весе или удаляются автоматически. Авторская репутация = функция от outcome utility всех его cards (PageRank на исходах, не на голосах).

### 16.8 Roadmap

| Фаза | Срок | Что |
|---|---|---|
| **v0** | 1 месяц | Opt-in profile + карта H3-hex плотности + ручной QR-обмен. Цель: проверить хочет ли это кто-то вообще. |
| **v1** | 3 месяца | BLE handshake AK-47 v2 + signed knowledge cards + RAG-буфер интеграция. |
| **v2** | 6 месяцев | Quest engine + outcome utility tracking + reputation на master-pubkey. |
| **v3** | 12 месяцев | Host events + travel quests + партнёрство с локальными touristboard / coworking / конфами. |

### 16.9 Open questions

1. **Liability.** Знакомства через LoRA Hunt → что-то пошло не так в реале — disclaimer + reporting tool обязательны.
2. **Cold start.** Первые 1000 юзеров — где они встречаются? Возможно seeded events в Тбилиси / Ереване / других city-hubs где Jippy уже есть.
3. **Verification of in-person claim.** BLE-радиус ~10m, но можно фармить через external BLE bridge? Тестирование в реальных условиях покажет.

### 16.10 Связь с остальной архитектурой

- §12.6 (Conservative weight updates) — правила промоута knowledge cards в веса
- §11 (Sensor pipeline) — ASR при live-exchange генерит cards из разговора
- `project_contribution_economy.md` — royalty при exchange-events
- `project_lora_guilds.md` — Phase 2 hierarchical routing над collected LoRA
- `FRONTIER_ATTACK.md` дыра #3, #5 — physical mesh как структурное преимущество над облаком

---

## 17. Memory Architecture — где живёт что и почему именно там

**Тезис:** это **архитектурный фундамент Jippy**. Все остальные разделы (sensors, tool use, LoRA Hunt, mesh, continuous learning) ссылаются сюда за «куда положить». Если этот раздел путается — путается всё.

### 17.1 Frame: memory taxonomy из нейропсихологии

Tulving / Squire разделили человеческую память на типы по функции и нейроанатомии. Та же схема ложится на нашу архитектуру **буквально**, не как метафора:

| Тип памяти у человека | Что | Наш слой |
|---|---|---|
| **Эпизодическая** | «вчера в кафе с Васей» | Слой 1: Native Search |
| **Семантическая** | факты, структурированное знание | Слой 2: Structured Knowledge (RAG) |
| **Процедурная + стилистическая** | «как ездить на велосипеде», «как я думаю» | Слой 3: LoRA |
| **Глубокая ре-консолидация** | сон, длительные изменения личности | Слой 4: Full Fine-tune |
| **Внешняя справка** | словари, библиотеки, википедия | Слой 0: World Knowledge |

Мы не изобретаем — повторяем разделение труда которое эволюция уже оптимизировала. Frontier-модели смешивают всё в одну параметризацию (отсюда галлюцинации фактов + плохая персонализация). Мы разделяем явно.

### 17.2 Обзор слоёв

```
┌────────────────────────────────────────────────────────────────┐
│ Слой 4 (Harness) — оркестрация: routing, permissions, triggers│
└────────────────────────────────────────────────────────────────┘
            ↓                 ↓               ↓              ↓
┌──────────────┐ ┌─────────────────┐ ┌──────────────┐ ┌──────────┐
│ Слой 0       │ │ Слой 1          │ │ Слой 2       │ │ Слой 3   │
│ WORLD        │ │ NATIVE SEARCH   │ │ STRUCTURED   │ │ LoRA     │
│ KNOWLEDGE    │ │ исходники       │ │ KNOWLEDGE    │ │ ПОВЕДЕНИЕ│
│              │ │                 │ │              │ │          │
│ Backbone +   │ │ TG, Chrome,     │ │ 5 подслоёв   │ │ Voice +  │
│ Web (Sonar)+ │ │ files, photos,  │ │ ~10-100 MB   │ │ multi-V +│
│ cached       │ │ contacts        │ │ SQLite + vss │ │ patterns │
│              │ │ нативные API    │ │              │ │ + onto-  │
│              │ │ + наш FTS5      │ │              │ │ logy     │
└──────────────┘ └─────────────────┘ └──────────────┘ └──────────┘

Слой 4: FT (full fine-tune) — РЕДКАЯ операция, серверная, milestone
   - Phase Г community backbone (federated averaging)
   - L10 personal graduation (раз в годы для активных юзеров)
```

### 17.3 Слой 0 — World Knowledge

**Что:** энциклопедическая справка о мире, не о юзере.

```
0a — Backbone parametric memory
    Qwen 0.8B что "знает" в весах. Базовая эрудиция.
    Дёшево, мгновенно, но галлюцинации + устаревание.

0b — Web search (online)
    Sonar API (live), Google search fallback.
    Свежие факты, current events. Только при сети.

0c — Curated cache (Phase Б+)
    Wikipedia subset (выбранные домены), специализированные базы.
    Офлайн-готово. Обновляется по расписанию.
```

**Routing rule:** если query про факт о мире и не про юзера — Слой 0. Если есть сеть — 0b предпочтительнее 0a (свежее). Если нет — 0c затем 0a.

### 17.4 Слой 1 — Native Search (episodic)

**Что:** прямой поиск по сырым данным на устройстве. Источник правды, без копирования.

| Платформа | Технология |
|---|---|
| iOS | Core Spotlight + NSMetadataQuery + Search framework + App Intents |
| Android | MediaStore + Room FTS5 + Accessibility Service + ContentResolver |

**Цель:** <100ms на запрос по 1M items.

**Источники инжеста в наш FTS5-индекс** (для приложений с закрытым sandbox):
- TG: import через export (Phase 0 валидирован — 691K msgs)
- Chrome/Safari: history API + bookmarks
- Email: IMAP/Gmail с консентом
- Custom apps: app-specific export или accessibility-скрейп

**Принцип:** индекс ~5% от исходника, копии данных — нет. Сырое только в одном месте (где приложение его уже хранит).

### 17.5 Слой 2 — Structured Knowledge (semantic)

**Что:** извлечённое + производное знание о юзере и его окружении. SQLite + sqlite-vss для embedding-индекса. Размер: 10-100 MB.

#### 17.5.1 Подслой 2a — User Profile (tiered, hypothesis-driven)

**Тезис:** не плоский набор фактов. Каждое поле живёт в одном из **трёх tier'ов** с явной confidence + provenance + механизмом активной проверки гипотез.

**Зачем не плоско:** старая v1 schema (`profile_json` свободной формы) смешивала «Тбилиси» (точно знаем) и «вероятно ценит честность» (домыслили) — без отметки качества. Это ведёт к hallucination (system выдаёт hypothesis как fact) и не даёт **активно** проверять то в чём не уверены. Marble (kg.js) показал что **hypothesis-driven подход** ускоряет cold start с месяцев до первой сессии.

##### Schema каждого поля

```python
profile_field = {
  "value": Any,                       # фактическое значение
  "tier": "fact" | "inference" | "hypothesis",
  "confidence": 0.0..1.0,
  "evidence": [event_id_1, event_id_2, ...],   # ссылки в Layer 1
  "synthesized_from": str | None,     # объяснение (для inference/hypothesis)
  "test_question": str | None,        # вопрос для активной проверки (для hypothesis)
  "last_confirmed": ISO8601_timestamp,
  "decay_after_days": int             # tier понижается если не подтверждается
}
```

##### Три tier'а

| Tier | Когда | Confidence | Decay | Использование |
|---|---|---|---|---|
| **fact** | Прямо сказано юзером ИЛИ ≥3 независимых evidence | 0.85-1.0 | Не decay'ится (но last_confirmed обновляется) | Передаётся в prompt как ground truth |
| **inference** | Выведено из ≤2 evidence или поведенческого паттерна | 0.5-0.85 | Через `decay_after_days` без подтверждения → tier понижается до hypothesis | Передаётся в prompt с пометкой «вероятно» |
| **hypothesis** | LLM-сгенерированное предположение из cross-field reasoning, evidence слабая или отсутствует | 0.2-0.5 | Через `decay_after_days` без подтверждения → удаляется | НЕ передаётся в prompt по умолчанию. Используется только для active testing. |

##### Поля профиля (примеры)

```python
profile = {
  "home_address": {
    "value": "Тбилиси",
    "tier": "fact",
    "confidence": 0.95,
    "evidence": ["ev_abc", "ev_def", "ev_ghi"],
    "last_confirmed": "2026-04-15T..."
    # decay_after_days отсутствует — fact не decay'ится
  },
  "current_projects": [
    {
      "value": "gpu.social",
      "tier": "fact",
      "confidence": 0.99,
      "evidence": [...10 event_ids...],
      "last_confirmed": "2026-04-29"
    },
    {
      "value": "Jippy mobile app",
      "tier": "inference",
      "confidence": 0.7,
      "evidence": ["ev_xyz"],
      "synthesized_from": "несколько упоминаний phone-side architecture",
      "decay_after_days": 30
    }
  ],
  "values_inferred": [
    {
      "value": "честность важнее гармонии",
      "tier": "inference",
      "confidence": 0.6,
      "evidence": ["ev_42"],
      "synthesized_from": "поведенческий паттерн в 3 disagreements",
      "decay_after_days": 60
    },
    {
      "value": "предпочитает работать утром",
      "tier": "hypothesis",
      "confidence": 0.3,
      "evidence": [],
      "synthesized_from": "статистика активности TG: 70% сообщений до 14:00",
      "test_question": "ты обычно лучше работаешь утром или ночью?",
      "decay_after_days": 14
    }
  ]
}
```

##### Pipeline извлечения — 3 tier'а в `extract_profile.py`

```
Tier 1 — Heuristic facts
  Источник: TG metadata + sensor data + явный onboarding
  Output: home_address, frequent_locations, languages, active_hours
  Все поля → tier="fact" если evidence ≥ 3, иначе → "inference"

Tier 2 — LLM extraction of explicit statements
  Источник: события где юзер прямо себя описывает («я живу в», «я работаю в»)
  LLM: Qwen 7B/32B, prompt с few-shot примерами
  Output: current_projects, languages, явные ценности
  Tier зависит от частоты упоминания + явности

Tier 3 — Hypothesis synthesis ⭐ (новое)
  Источник: все Tier 1+2 facts + cross-field paterns
  LLM получает: "Вот что мы знаем о юзере. Сгенерируй 5-10 гипотез про
                него которые не упомянуты явно но логически следуют."
  Output: values_inferred[hypothesis], routine_patterns[hypothesis], ...
  Все → tier="hypothesis", confidence 0.2-0.4
  Каждая получает test_question — что спросить чтобы подтвердить
```

##### Active hypothesis testing

Harness'у новая ответственность: **проактивно проверять hypothesis-tier поля**.

**Trigger conditions:**
- Юзер задаёт открытый/общий вопрос (low specificity) — это окно для side-testing
- Прошло >7 дней без любых profile updates
- Hypothesis с lowest confidence есть в очереди

**Поведение:**
```
[юзер пишет общий вопрос]
[harness отвечает на вопрос как обычно]
[в конце ответа добавляет — non-blocking — test question]:
  "...кстати, я заметил что ты часто пишешь рано утром.
   Это твоё лучшее время для работы или просто привычка?"
```

**Юзер отвечает:**
- Подтверждает → tier promotion (hypothesis → inference → fact), confidence growths, evidence добавляется
- Опровергает → tier=rejected (history сохраняется), удаляется из active profile
- Игнорирует → нейтрально, hypothesis остаётся, decay timer тикает

##### Decay механизм

Раз в день (или при каждом profile read) — фоновая проверка:

```python
for field in profile.fields_with_decay:
  age_days = (now - field.last_confirmed).days
  if age_days > field.decay_after_days:
    if field.tier == "fact":
      pass  # facts не decay'ятся
    elif field.tier == "inference":
      field.tier = "hypothesis"
      field.confidence *= 0.5
    elif field.tier == "hypothesis":
      profile.archive(field)  # удаляется из active, остаётся в history
```

##### Cross-field reasoning chains

Hypothesis synthesis (Tier 3) использует **связи между полями** для генерации, не одно поле в вакууме:

```
Известно: home="Тбилиси" + projects=["gpu.social"] + Vadim в close_circle + topic="GPU rentals" частый
   ↓ LLM cross-field reasoning
Гипотеза: "Yuka — co-founder gpu.social с Vadim как backend partner,
          основной revenue plan — GPU rental marketplace"
   tier="hypothesis", confidence=0.7
   test_question: "Vadim твой co-founder в gpu.social?"
```

Это работает в связке с Layer 2c (Knowledge Map): hypothesis может ссылаться на концепты в графе.

##### Использование в harness

При assemble prompt контекста:

```python
def assemble_profile_context(user_id):
    profile = layer2a.get_profile(user_id)
    facts = [f for f in profile if f.tier == "fact"]
    inferences = [f for f in profile if f.tier == "inference"]
    # hypotheses НЕ передаются в основной prompt
    
    return {
      "facts": format_facts(facts),                    # "User lives in Tbilisi."
      "inferences": format_inferences(inferences),     # "User likely prefers honesty over harmony."
    }
```

Затем system prompt: **«Use facts as ground truth. Use inferences with appropriate hedging ("probably", "likely"). Never invent details not provided.»**

##### Privacy и шифрование

Schema хранится в `user_profile` table в SQLCipher-encrypted brain.db. Поля типа `phase_of_life`, `values_inferred` — sensitive, никогда не покидают устройство без явного opt-in.

##### Связь с другими разделами

- §12 (Continuous Learning) — корректировки юзера → updates evidence + tier promotion
- §17.5.4 (Operational layer 2d) — corrections table содержит все promote/reject events для audit
- §17.5.3 (Knowledge Map 2c) — cross-field reasoning подтягивает концепты из графа
- §10 (Tool Use) — Jippy может задать test question через native UI prompt
- `JIPPY_BACKEND_USER_TODO.md` — реализация передаётся разработчику

##### Eval queries для tier system

`backend/eval/queries.jsonl` должны включать:
- Cold-start scenarios — после N сообщений сколько hypotheses synthesized?
- Promotion path — за сколько дней hypothesis → inference при подтверждениях?
- Decay correctness — устаревшая hypothesis действительно архивируется?
- Hallucination test — hypotheses никогда не передаются как facts в prompt?

##### UI требование «Кто я» (часть продукта)

Profile **должен быть видимым и редактируемым** через UI. Не админка — это часть продукта. Без UI юзер не может ни увидеть гипотезы, ни подтвердить, ни поправить пол если автоопределение ошиблось.

**Структура — 3 секции (по tier'ам):**

```
┌─────────────────────────────────────────────────┐
│ 🟢 ФАКТЫ                                        │
│ ─────────────                                   │
│ Дом:        Тбилиси                  [edit]     │
│ Проект:     gpu.social               [edit]     │
│ Язык:       русский                  [edit]     │
│ Пол:        мужской                  [edit]     │
│ Местоимения: он/его                  [edit]     │
│                                                 │
│ 🟡 ВЕРОЯТНО                                     │
│ ─────────────                                   │
│ Ценности:   честность важнее гармонии           │
│             [✓ да] [✗ нет] (видно в 3 разговорах)│
│                                                 │
│ 🟣 ДОГАДКИ                                      │
│ ─────────────                                   │
│ ❓ Ты обычно работаешь лучше утром, верно?      │
│    [✓ да] [✗ нет] [через неделю]                │
│                                                 │
│ + Добавить факт о себе вручную                  │
└─────────────────────────────────────────────────┘
```

Действия юзера → API endpoints (которые **уже реализованы** в backend):
- Confirm hypothesis → `POST /jippy/brain/profile/confirm` (tier promotion)
- Reject hypothesis → `POST /jippy/brain/profile/reject` (archive)
- Edit fact → `POST /jippy/brain/profile/set` (manual override)
- View all → `GET /jippy/brain/profile`
- View hypotheses queue → `GET /jippy/brain/profile/hypotheses`

UI — это **только frontend работа**, backend готов.

#### 17.5.2 Подслой 2b — Relational Graph

```
Узлы: люди в close circle (~20-50)
Атрибуты узла:
  contact_info: { phones, emails, addresses, telegram }
  relationship_type: mom | partner | co-founder | friend | work
  voice_samples: [5-20 representative reply pairs]
  topics_discussed: [tags юзер обсуждает с этим человеком]
  topics_avoided: [явные anti-topics]
  whitelist_for_§12.6: bool
  last_interaction: timestamp
  closeness_score: 0..1                    # multi-signal computed score
  closeness_tier: close | warm | acquaint | transactional   # auto + user-overridable
  user_override: bool                      # 1 если юзер вручную выставил tier

Рёбра: связи между людьми (если знают друг друга),
       тип связи (близкие, коллеги, и т.д.)
```

Это **топология социального мира юзера** как первичный объект. Используется для:
- Multi-voice routing (см. 17.6)
- §12.6 close-circle whitelist (правила conservative weight updates)
- LoRA Hunt mesh trust

##### Closeness multi-signal score

**Правило:** **частота сообщений ≠ близость.** Колл-центр пишет чаще чем мама. Старый друг которого видишь раз в год — ближе чем коллега из чата.

`closeness_score` считается как weighted sum по сигналам:

```python
closeness_score = weighted_sum(
    reply_latency_score,      # как быстро ты ему отвечаешь (быстро = ближе)
    self_initiated_ratio,     # % сообщений где ТЫ начал, не он
    emotional_content,        # личные/эмоциональные темы vs транзакционные
    bidirectionality,         # равный обмен vs один пишет, другой "ок"
    time_of_day_personal,     # ночные/личные часы vs только рабочие
    avg_message_length,       # длинные осмысленные vs короткие функциональные
    topic_diversity,          # обсуждаете много vs одну тему
    persistence_years,        # годы общения vs пара недель
)
```

`closeness_tier` маппится из score:
- `close` (≥0.75) — ближний круг
- `warm` (0.5-0.75) — тёплые
- `acquaint` (0.25-0.5) — знакомые
- `transactional` (<0.25) — функциональные контакты

**User override:** через UI юзер может перетащить любого человека между tier'ами. `user_override=true` фиксирует, automatic re-scoring **не перезаписывает** override.

##### UI требование (часть продукта, не админка)

Юзер должен видеть и **управлять** social graph'ом. Минимум:

**Kanban-style 4 колонки** — близкий круг / тёплые / знакомые / транзакционные. Drag-and-drop между колонками = manual override `closeness_tier`.

На каждом person-card:
- Display name + photo + recent voice samples (preview)
- Toggle `whitelist_for_correction` (§12.6 trusted) — checkbox
- Edit relationship_type (mom/partner/co-founder/friend/work)
- Show topics_discussed (tags)
- Hide / archive person
- See closeness_score breakdown (какие сигналы вытащили его наверх)

#### 17.5.3 Подслой 2c — Knowledge Map (Obsidian-style) ⭐

**Самый новый и недооценённый компонент.** Топология ума юзера как граф.

```
Узлы: концепты (теги)
Атрибуты узла:
  name, level (если LoRA Hunt tag), embedding,
  pointers: [Слой1_source_ids] — где живёт content
  pinned_to_core: bool                  # юзер пометил концепт важным (boost weight)
  user_hidden: bool                     # юзер скрыл (мусор)
Рёбра: связи между концептами как видит их ЭТОТ юзер
Атрибуты ребра:
  type: is-a | related-to | contradicts | causes | example-of
  strength: 0..1 (частота со-упоминания)
  user_explicit: bool (юзер сам провёл связь, не извлечена)
```

##### Extraction — обязательно через NER, не frequency-only

**Правило:** v1 implementation использовал частотный фильтр со stopword-листом → топ концептов получались «плиз», «чтобы», «было». Это **degraded mode**, не production.

**Production extraction:**

```python
# 1. spaCy multilingual NER
import spacy
nlp = spacy.load("xx_ent_wiki_sm")  # или ru_core_news_sm для RU
# Только NER-типы попадают в концепты:
#   PERSON / LOC / GPE / ORG / PRODUCT / EVENT / WORK_OF_ART

# 2. PoS теггинг — только NOUN/PROPN, никаких VERB/PART/ADV/ADP

# 3. TF-IDF importance — отсев слишком частых (которые проскочили stopwords)

# 4. Min frequency ≥ 3 — отсев слишком редких

# 5. Aliases extraction — Швеция/Швецию/Швеции/Sweden → один canonical concept
#    Используется fuzzy match (Levenshtein < 3) + lemma normalization
#    Aliases хранятся в concept_aliases table

# 6. (опционально, Phase Б) Hierarchical clustering через GraphRAG
#    Швеция → группа «Скандинавия» → группа «Европа»
```

**Frequency-based fallback** оставляем только для languages где spaCy не покрывает (грузинский, армянский). Default — NER pipeline.

##### Pipeline извлечения (с NER)

```
TG msgs / conversations / files → batch nightly →
  spaCy NER + PoS → only PERSON/LOC/ORG/PRODUCT/PROPN nouns →
  alias merge (canonicalization) →
  store в SQLite nodes+edges + concept_aliases →
  embed nodes для semantic lookup (mxbai-embed) →
  expose как knowledge_map.query(concept, depth=2)
```

**Алгоритм Phase Б:** Microsoft GraphRAG (community detection + hierarchical summarization).

**Зачем:** разные графы для разных людей = персонализация на уровне ontology, не стиля. Когда harness отвечает упоминая концепт X, он запрашивает граф: «упомянул X → какие концепты юзер обычно с ним связывает → подтянуть их в контекст». Ответ чувствуется «по-моему», не «по-чьему-то».

**Frontier этого не имеет by design** — у них один граф для всех.

##### UI требование (часть продукта)

Юзер должен видеть и **управлять** своей картой ума. Минимум:

**Force-directed graph** через `react-force-graph` или `cytoscape.js`:
- Узел = концепт; **размер** = importance score; **цвет** = NER-тип (person/location/project/...)
- Рёбра = связи; **толщина** = strength; **цвет** = тип (is-a / related-to / contradicts)

Действия юзера:
- Кликнуть концепт → expand neighbors (depth+1)
- **Скрыть/удалить** (видно «плиз»? удали — `user_hidden=true`)
- **Объединить дубликаты** вручную (если NER не справился)
- **Pin to core** — пометить концепт важным (`pinned_to_core=true` → boost retrieval weight)
- **Tag** — добавить свой тег (для LoRA Hunt)
- Поиск по карте (zoom to node)
- **Add edge** — провести связь вручную («эти два концепта связаны для меня»)

Скрытое юзером в `user_hidden=true` — никогда не возвращается в retrieval, но не удаляется (юзер может вернуть).

#### 17.5.4 Подслой 2d — Operational Layer

```
§12.1 Correction log (мои ошибки + правки юзера)
§12.6 Knowledge cards (от mesh, LoRA Hunt, с провенансом)
Errors+lessons (provenance (a) — личный опыт)
Open questions (повторяющиеся темы за 6+ мес)
Tags+levels из §16 (LoRA Hunt identity, gранулярный opt-in)
```

#### 17.5.5 Подслой 2e — Anti-patterns

Negative space — что юзер НЕ делает / НЕ говорит / НЕ верит.

**Распределение по слоям** (это «explicit register», реальная защита распределена):

| Тип antipattern | Где живёт |
|---|---|
| Hard rules («никогда не делиться тегом X») | Harness logic (Слой 4) |
| Стилистические («не пишу best regards») | LoRA negative training (DPO) |
| Фактические («не верю что AGI скоро») | RAG 2e |
| Поведенческие («не отвечаю на работу после 21:00») | RAG 2e + harness фильтр |

### 17.6 Слой 3 — LoRA (procedural + stylistic)

**Правило:** LoRA хранит **knowing-how**, не **knowing-that**.

| LoRA ✅ | LoRA ❌ (это в Слой 1-2) |
|---|---|
| Voice (как пишет) | Имена, даты, числа |
| Multi-voice (по контексту собеседника) | «Вадим = backend на сервере X» |
| Decision patterns | Конкретные события |
| Conceptual ontology (паттерны связей) | Адреса, телефоны |
| Эстетика | Точные цитаты |
| Anti-style | Технические спеки |

**Мнемоника:** «если ошибка в этом месте незаметна — LoRA. Если заметна — RAG». Стиль ошибётся на 5% — нормально. Имя ошибётся на 5% — катастрофа.

**Multi-voice реализация (гибрид):**
- **A) Speaker token conditioning** — для всех контактов. Префикс `<to:mom>` в обучающих данных, harness вставляет на инференсе. Один LoRA.
- **B) Per-relationship мини-LoRA** — для top 3-5 ближайших (mom, partner, и т.д.). ~5MB каждая, активируются Arrow routing'ом.
- **C) RAG few-shot** — для редких контактов. Voice samples из 2b как few-shot examples. Не трогает веса.

### 17.7 Слой 4 — Full Fine-tune (consolidation)

**Правило:** на телефоне в v1 FT **не делаем вообще**. Это серверная или milestone операция.

#### 17.7.1 Когда FT нужен

| Trigger | Сценарий |
|---|---|
| LoRA capacity saturated | >10M training examples, rank 64+ деградирует на val |
| Community averaging | Phase Г: agg LoRA-deltas от тысяч юзеров → community backbone |
| L10 milestone | Юзер дошёл до уровня 10 — server FT всего стека → personal backbone (1.5 GB) |

#### 17.7.2 Phase Г — серверный community FT

**Главный кейс для FT в нашей архитектуре.**

```
Раз в месяц на сервере:
  → Собираем (с opt-in, anonymized) LoRA-deltas от тысяч юзеров
  → Дистиллируем в community-LoRA через averaging + filter
  → Полный FT backbone'а на community-LoRA + open-source data
  → Новый "community Qwen v2" пушится всем юзерам как обновлённый base
  → Их personal LoRAs остаются поверх, не теряются
```

**Эффект:** backbone становится умнее на коллективной мудрости, никаких личных данных юзера он не видит напрямую. Это **общая библиотека человечества** — механизм transmissions знаний между поколениями юзеров (как письменность у людей).

#### 17.7.3 L10 Personal FT — milestone ceremony

Маркетинговая дуга «твой AI окончил школу»:

```
Year 0: чистый Qwen 0.8B + LoRA 14 MB
Year 2: Qwen + 47 LoRAs в стеке (voice, домены, multi-voice)
Year 3: достигнут L10 → server FT всего стека →
        персональный backbone «Yuka-Qwen v1» (1.5 GB) →
        можно скачать, владеть, форкнуть, продать (contribution economy)
```

Техническая необходимость (LoRA-стек устаёт) **+** продуктовая церемония (ритуал градуации).

#### 17.7.4 Когда FT НЕ делать

- Каждые N дней «на свежих данных» — это **вреднее** чем LoRA. Catastrophic forgetting.
- Для персонализации новой темы — это работа LoRA, не FT.
- На телефоне — никогда (compute не вытянет).

### 17.8 Routing — как Harness принимает решения (v2 — parallel fetch)

**Принцип v2:** не классифицируем intent через regex и не лезем в один источник. **Параллельно опрашиваем все локальные слои** (Layer 1+2), отдаём найденное в LLM как rich context, **LLM сам разбирается** что использовать через семантику данных.

**Зачем v2:** regex-классификатор не отличает «Семих» (друг) от «Эйнштейн» (мировой факт) от «Лайка» (кличка собаки в концептах). Parallel fetch + LLM-as-router снимает классификацию с нашего кода и делегирует семантическое понимание модели — там где она сильна.

```
incoming query
    │
    ▼
┌──────────────────────────────────────────────────────────────────┐
│ 1. Query Rewriter (LLM #1, Cerebras small)                      │
│    Conversational query → {search_query, sender_hint, self_only}│
│    Resolves pronouns from chat history (см. §17.8.2)            │
└──────────────────────────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────────────────────┐
│ 2. PARALLEL FETCH всех local Layer 1+2 источников (~100ms)      │
│    asyncio/threadpool — все одновременно:                       │
│      ├─ Layer 1 (FTS5 search) с sender_filter                   │
│      ├─ Layer 2a (Profile) — facts + inferences (НЕ hypotheses) │
│      ├─ Layer 2b (People) — alias lookup для всех имён в query  │
│      ├─ Layer 2c (Concepts) — concept + neighbors depth 1       │
│      └─ Layer 2e (Anti-patterns) — релевантные red flags        │
│    Все находки → в context bundle (с явными section headers)    │
└──────────────────────────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────────────────────┐
│ 3. CONDITIONAL Layer 0 fetch                                    │
│    has_temporal_marker? (сегодня / новости / курс / погода / now)│
│      YES → Layer 0b (Sonar web search)                          │
│      NO  → skip (Cerebras parametric memory сам справится)      │
└──────────────────────────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────────────────────┐
│ 4. ADAPTIVE chat history compression (см. §17.8.1)              │
│    if tokens(history) ≤ 10K: send ALL                           │
│    else: summary(older) + last_2_turns_verbatim + current       │
└──────────────────────────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────────────────────┐
│ 5. Format prompt с rich context                                 │
│    System: "User is {gender}, speaks {lang}. Use FACTS as       │
│             ground truth. Use INFERENCES with hedging. Never    │
│             invent details. If information is missing, say so." │
│    Context bundle (явные секции — LLM умеет в них ориентироваться):│
│      === PROFILE FACTS ===                                      │
│      === PROFILE INFERENCES (probably) ===                      │
│      === RELEVANT PEOPLE ===                                    │
│      === RELEVANT CONCEPTS ===                                  │
│      === RELEVANT MEMORIES (Layer 1) ===                        │
│      === WORLD FACT (Layer 0b, if any) ===                      │
│      === ANTI-PATTERNS (avoid) ===                              │
└──────────────────────────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────────────────────┐
│ 6. Backbone call (LLM #2, multi-tier fallback)                  │
│    Cerebras 70B → Cerebras Paid → Groq                          │
│    SSE streaming → токены льются в UI                           │
│    LLM сам решает что из rich context использовать через        │
│    semantic relevance (это и есть «router as LLM»)              │
└──────────────────────────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────────────────────┐
│ 7. Log inference (бортовой самописец)                           │
│    inference_log: query, layers_used, latency_ms, response       │
└──────────────────────────────────────────────────────────────────┘
```

**Что убрано в v2:**
- ❌ Regex `_INTENT_PATTERNS` — больше не нужны
- ❌ Per-intent `_assemble_*` функции — заменены одним parallel fetch
- ❌ Hard rules check (отдельно убран — см. note ниже)
- ❌ Always-call Sonar — теперь только при temporal markers

**Что добавлено:**
- ✅ Concurrent fetch всех Layer 1+2
- ✅ Adaptive context compression
- ✅ Layer 0 split: 0a (parametric default, free) + 0b (Sonar conditional)
- ✅ LLM-as-router через semantic context

**Принцип:** **поиск дешевле классификации.** Local fetches параллельные ~100ms, parametric memory backbone бесплатна. Платим только когда реально нужен Sonar (temporal). Качество выше — LLM не промахивается на классификации потому что её просто нет.

#### 17.8.1 Adaptive context window

Вместо hardcoded last-N сообщений — token-budgeted compression:

```python
def build_messages_context(chat_history, current_msg, budget_tokens=10000):
    if count_tokens(chat_history) <= budget_tokens:
        return chat_history + [current_msg]   # всё помещается verbatim

    # Не помещается → split + compress
    last_2_turns = chat_history[-4:]   # последние 2 Q+A пары
    older = chat_history[:-4]

    summary = summarize_with_llm(older)   # Cerebras call ~500ms

    return [
        {"role": "system", "content": f"Earlier in this conversation: {summary}"},
        *last_2_turns,
        current_msg
    ]
```

**Параметры (default):**
- `budget_tokens = 10000` — буфер до Cerebras 32K context limit
- `last_2_turns = 4 messages` — гарантированный verbatim recent context
- Edge case (диалог >100K) — иерархическая компрессия (Phase Б)

#### 17.8.2 Pronouns resolution в Query Rewriter

Rewriter получает текущий вопрос + chat history → возвращает:
```json
{
  "search_query": "адрес",
  "sender_hint": "Виталик",
  "self_only": false,
  "resolved_pronouns": {"он" → "Виталик"}
}
```

Если current message не имеет местоимений — rewriter может работать только на самом current message. Если есть — обязательно chat history.

#### 17.8.3 Failure modes

- Layer 1 пуст → ничего страшного, LLM получит другие слои
- Layer 2 пуст → degraded personalization, LLM может использовать parametric
- Backbone unavailable → graceful error, не crash
- Sonar 5xx → fallback на backbone parametric
- Все слои дали 0 results → честный «у меня нет информации по этому вопросу»

**Note (2026-04-29):** ранний step «hard_rules check» убран из routing. Будет возвращён в Phase Б когда появится UI для управления правилами. До этого моделирование запретов через system prompt инструкции.

### Backbone — текущее состояние и roadmap

**Сейчас (v1):** все LLM-вызовы (query rewriter, hypothesis synthesis, main answer) идут через **Cerebras 70B → Cerebras Paid → Groq fallback** (multi-tier). Driven by `JIPPY_BACKBONE_TIERS` env. Latency p50 ~4.3 сек end-to-end.

**Roadmap (Phase 4+):** когда personal LoRA готова и интегрирована — переключение **с Cerebras на on-device Qwen 0.8B + LoRA** для:
- Voice generation (Layer 3 task) — обязательно через LoRA
- Personal knowledge expansion — через LoRA
- Privacy-sensitive queries — через LoRA

Cerebras остаётся для:
- Query rewriter (нужна большая модель для понимания pronouns + контекста)
- Hypothesis synthesis (Tier 3, нужна большая модель для cross-field reasoning)
- World facts (если Sonar ниже качества)

**Switch criteria:** LoRA готова + on-device latency <2 сек + voice match score ≥80% (см. eval).

### 17.9 Cheat sheet — что куда

| Тип данных | Слой |
|---|---|
| Сообщения, события, файлы | 1 |
| Адреса (свои) | 2a |
| Адреса (друзей) | 2b |
| Имена, контакты | 2b |
| Концепты + связи между ними | 2c |
| Корректировки от юзера | 2d |
| Knowledge cards от mesh | 2d |
| LoRA Hunt теги + уровни | 2d |
| Open questions | 2d |
| Hard rules / запреты | 4 (harness) + 2e |
| Стилистические anti-patterns | 3 (LoRA negative) + 2e |
| Voice (общий) | 3 |
| Voice по человеку | 2b (samples) + 3 (multi-voice LoRA) |
| Decision patterns | 3 |
| Aesthetic preferences | 3 |
| Факты о мире (свежие) | 0b |
| Факты о мире (офлайн) | 0c → 0a |
| Эрудиция базовая | 0a (backbone parametric) |

### 17.10 Что это даёт против frontier

| Аспект | Frontier | Jippy |
|---|---|---|
| Где данные юзера | их облако | его устройство |
| Дублирование raw data | RAG-копия в их облаке | нет, native search |
| Граф концептов юзера | нет (один на всех) | per-user (2c) |
| Multi-voice по людям | нет | 2b + 3 |
| Anti-patterns | нет explicit register | 2e + harness |
| World facts свежие | их облако | Слой 0 (web/cache/backbone) |
| Полный FT под юзера | экономически невозможно | Слой 4 milestone |

**Главное:** их архитектура не может сжаться до телефона без переизобретения с нуля. У нас она уже там.

### 17.11 Связь с остальной спекой

- §3 (Компоненты) — архитектурная схема в целом, §17 уточняет memory-часть
- §5 (Хранение) — формат на диске, §17 определяет логические слои
- §10 (Tool Use) — вызывается Harness'ом из Слоя 4
- §11 (Sensor Pipeline) — кормит Слой 1 + извлекает в 2a/2c
- §12 (Continuous Learning) — обновление Слоя 3 через 2d
- §12.6 (Conservative weight updates) — правила промоута 2d → 3
- §13 (Form Factors) — каждая surface использует ту же memory-архитектуру
- §14 (Sync) — синхронизация между surfaces работает на уровне Слоёв 2-3 (не 1)
- §16 (LoRA Hunt) — обмен knowledge cards = расширение 2d
- `FRONTIER_ATTACK.md` — дыры #1, #2, #5 атакуются именно через эту memory-архитектуру

---

## 18. Backlog — feature-идеи на парковке

**Назначение:** место для product/feature идей которые **не реализовываем сейчас** но не хотим потерять. Не планы, не решения — парковка.

**Что сюда:**
- Конкретные feature-идеи (новые экраны, режимы, integrations)
- Use cases для существующих компонентов (sensor pipeline, tool use, etc.)
- Расширения существующих разделов

**Что НЕ сюда** (ходит в `FRONTIER_ATTACK.md` Strategic Backlog):
- Новые архитектурные дыры frontier
- Sequencing tweaks
- Маркетинговые / позиционные идеи

**Жизненный цикл идеи:**
1. **parked** — закинули, ещё не оценивали
2. **queued** — оценили, ждёт окно реализации
3. **promoted-to-§X.Y** — превратилась в спеку, перенесена в нужный раздел
4. **rejected** — оценили, решили не делать (с обоснованием, не удаляем)

При promote — идея **перемещается** в нужный раздел спеки, а в §18 остаётся one-liner: «promoted to §X.Y on YYYY-MM-DD». История не теряется.

**Формат записи:**

```markdown
### Idea N — <Название>

**Добавлена:** YYYY-MM-DD
**Status:** parked | queued | promoted-to-§X.Y | rejected
**Связь:** §X (раздел), дыра #N в FRONTIER_ATTACK
**Описание:** 2-5 предложений что это и зачем

**Что нужно чтобы оценить:**
- [ ] вопрос 1
- [ ] вопрос 2

**Что нужно чтобы построить:**
- зависимость 1
- зависимость 2
```

---

### Idea 1 — Mindful Cart

**Добавлена:** 2026-04-28
**Status:** parked
**Связь:** §11 (Sensor Pipeline), §17.5.3 (Knowledge Map), дыра #5 (grounding) в FRONTIER_ATTACK

**Описание:** Jippy в магазине **в момент выбора** даёт юзеру данные о нём же — что он покупал, что ел, как себя чувствовал после. Тело = робот, еда = программа. Лампочка в магазине = захват точки контроля над тем что станет твоим телом через 2-7 дней.

**Сценарий:**
```
[Юзер в Spar, сканирует круассан или просто кладёт в корзину]
Jippy: "За последний месяц 4 круассана. После них:
        средний sleep score -12%, утренняя энергия -18%.
        Хочешь продолжить или переключиться?"
```

**Почему это структурно невозможно для frontier:**
- Cloud AI не знает что юзер ел и как чувствовал
- Apple Health знает данные но не интерпретирует в контексте покупок
- Никто не связывает «решение о покупке → биологический эффект через N дней»

У нас все компоненты **уже есть** в архитектуре:
- §11 (Sensor): камера сканирует чек / штрихкод → list of items
- Слой 1: история покупок + история ощущений (sleep, mood, energy from Watch)
- Слой 2c (Knowledge Map): связи «когда я ел X → через Y дней N в энергии»
- Слой 3 (LoRA): стиль рекомендаций себе
- Harness: логика «ты в Spar, последние 3 раза когда брал X — спал плохо»
- §10 (Tool Use): Apple Health intent для pull данных

**Что нужно чтобы оценить:**
- [ ] Сколько юзеров реально хотят чтобы AI комментировал их корзину (UX research)
- [ ] Какая минимальная история нужна для useful insight (сколько недель данных)
- [ ] Privacy boundary: согласен ли юзер на link «здоровье ↔ покупки» в одной БД
- [ ] Не превратится ли это в anxiety-генератор (constant nagging)
- [ ] Что делать с социальным контекстом (партнёр купил, а insight выскочил тебе)

**Что нужно чтобы построить:**
- §11 Sensor Pipeline активен и собирает биометрию (Watch, sleep, mood логи)
- §10 Tool Use интеграция с Apple Health / Google Fit
- Receipt OCR / barcode scanner модуль (отдельный компонент)
- Food→nutrient database (open: USDA, OpenFoodFacts)
- Корреляционный engine в Слое 2c (статистически значимые временные связи)
- UX: в каком моменте показывать (в магазине? после чека? в дашборде?)

**Маркетинговая позиция:**
> «Твой ИИ знает что ты ешь и что с тобой происходит после. Локально, никто другой не узнает.»

**Связь с другими feature:**
- LoRA Hunt §16 — теги «nutrition», «biohacking» — обмен с экспертами в LoRA Hunt
- §12 Continuous Learning — корректировки от юзера («нет, после X я хорошо спал, у тебя ошибка в данных») промоутятся в Слой 2c

---

### Idea 2 — `why_text` на edges в Knowledge Map

**Добавлена:** 2026-04-29
**Status:** parked
**Связь:** §17.5.3 (Knowledge Map)
**Источник идеи:** AlexShrestha/marble (kg.js)
**Описание:** расширить `concept_edges` полем `why_text` — короткое объяснение **почему** связь существует, не только **что она есть**. Помогает explainability ответов («я связал X с Y потому что в марте ты сам провёл связь в разговоре с N»). Не меняет архитектуру, добавляет 1 поле в schema + extraction logic пишет объяснение при создании edge'а.

---

### Idea 3 — Multi-lens parallel evaluation в harness

**Добавлена:** 2026-04-29
**Status:** parked
**Связь:** §17.8 (routing rules), `project_lora_guilds.md`
**Источник идеи:** AlexShrestha/marble (swarm.js — 6 specialized lenses)
**Описание:** альтернативная модель routing'а — вместо one-shot intent classifier'а, N специалистов параллельно оценивают запрос с разных углов и голосуют. Концептуально близко к LoRA Guilds (Phase 2 hierarchical Arrow routing), но на уровне prompting, не LoRA. Возможные «lenses»: episodic / semantic / relational / temporal / contrarian. Когда созреет идея эволюции harness'а — всплывёт.
