# Feature Spec: Import AI History

**Feature:** Импорт истории из ChatGPT / Claude / Gemini  
**Priority:** P0 (critical path — решает cold start)  
**Owner:** TBD  
**Status:** Draft v2  

---

## 1. Проблема

Пользователь устанавливает Jippy. У него пустая модель которая ничего о нём не знает. Чтобы Jippy стал "своим", нужно накопить данные — но первые дни/недели он бесполезен. Пользователь уходит.

При этом у пользователя УЖЕ ЕСТЬ сотни/тысячи диалогов в ChatGPT, Claude или Gemini.

## 2. Решение

Два режима импорта — Light и Full — чтобы покрыть все сценарии от "лень возиться" до "хочу импортировать всё".

---

## 3. Два режима импорта

### LIGHT MODE — "Быстрый старт" (30 секунд)

**Суть:** Пользователь просит ChatGPT/Claude перечислить всё что он помнит → копирует текст → вставляет в Jippy.

**Что получаем:** User profile (профессия, интересы, стиль, язык, предпочтения). НЕ получаем историю диалогов.

**User Flow:**

```
Экран 1: "Быстрый старт"
┌─────────────────────────────────┐
│                                 │
│  Скопируй память из ChatGPT    │
│  за 30 секунд                   │
│                                 │
│  1. Откройте ChatGPT            │
│  2. Напишите:                   │
│  ┌─────────────────────────┐    │
│  │ Перечисли всё что ты    │    │
│  │ помнишь обо мне.        │    │
│  │ Выведи полностью.       │    │  
│  │              [Копировать]│    │
│  └─────────────────────────┘    │
│  3. Скопируйте ответ           │
│  4. Вставьте ниже:             │
│                                 │
│  ┌─────────────────────────┐    │
│  │                         │    │
│  │  [Вставить сюда...]     │    │
│  │                         │    │
│  │                         │    │
│  └─────────────────────────┘    │
│                                 │
│  ┌─────────────────────────┐    │
│  │     Готово →             │    │
│  └─────────────────────────┘    │
│                                 │
│  Хочу импортировать ВСЮ         │
│  историю → Полный импорт       │
│                                 │
└─────────────────────────────────┘
```

**Обработка (на сервере):**
1. Текст → LLM extraction → structured user profile JSON:
```json
{
  "profession": "маркетолог",
  "experience_years": 5,
  "interests": ["data analytics", "growth hacking", "Python"],
  "language": "ru",
  "style_preferences": ["детальные ответы", "с примерами"],
  "location": "Israel",
  "tools": ["Google Analytics", "Tableau", "Amplitude"],
  "raw_text": "...оригинальный текст для хранения..."
}
```
2. Profile → permanent storage
3. Raw text → permanent storage (принцип: храни всё)
4. Готово — Jippy использует profile для персонализации с первого вопроса

**Время обработки:** <5 секунд

---

### FULL MODE — "Полный импорт" (5-10 минут)

**Суть:** Пользователь экспортирует ZIP из ChatGPT/Claude/Gemini → загружает в Jippy → получает всю историю + profile.

**Что получаем:** Полная история диалогов (Q&A пары) + User profile + тематическая разметка.

**Проблема:** Экспорт доступен только через веб (десктоп/мобильный браузер), ZIP приходит на email. Пользователь на телефоне.

**User Flow — вариант A (телефон):**
```
1. Юзер в Jippy нажимает "Полный импорт"
2. Видит инструкцию:
   "Откройте ChatGPT в браузере телефона → Settings → Export"
3. Ссылка на email приходит
4. Юзер скачивает ZIP на телефон
5. Юзер нажимает "Загрузить ZIP" в Jippy
6. iOS: Document Picker / Android: File Picker → выбирает ZIP
7. ZIP → сервер на обработку
8. Прогресс-бар в приложении
9. Готово
```

**User Flow — вариант B (десктоп → телефон):**
```
1. Юзер экспортирует ZIP на десктопе
2. Заходит на jippy.ai/import в браузере десктопа
3. Загружает ZIP через web upload
4. Привязка к аккаунту по QR-коду из приложения
5. Обработка на сервере
6. Данные появляются в приложении автоматически
```

**User Flow — вариант C (Share Extension):**
```
1. Юзер скачивает ZIP на телефон (из email)
2. Нажимает "Поделиться" → выбирает Jippy
3. Jippy принимает ZIP через Share Extension
4. ZIP → сервер → обработка
```

---

## 4. Источники данных

| Платформа | Как экспортировать | Формат ZIP | Что внутри |
|-----------|-------------------|------------|------------|
| **ChatGPT** | Settings → Data Controls → Export | `conversations.json` + `chat.html` | Дерево: `mapping` → `parent`/`children`, роли: `user`/`assistant` |
| **Claude** | Settings → Privacy → Export Data | `conversations.json` (может быть `.dms` расширение) | Плоский массив `chat_messages`, роли: `human`/`assistant` |
| **Gemini** | takeout.google.com → Gemini Apps | JSON per conversation в `Takeout/Gemini Apps/` | Роли: `user`/`model`, формат нестабилен |

### Инструкции в приложении (в каждой карточке)

**ChatGPT:**
1. Откройте chatgpt.com → Settings (gear icon)
2. Data Controls → Export Data → Confirm
3. Проверьте email (ссылка действует 24 часа)
4. Скачайте ZIP → загрузите в Jippy

**Claude:**
1. Откройте claude.ai → Settings (инициалы внизу слева)
2. Privacy → Export Data
3. Проверьте email → скачайте ZIP (может быть .dms)
4. Загрузите в Jippy

**Gemini:**
1. Откройте takeout.google.com
2. "Deselect all" → найдите "Gemini Apps" → галочка
3. Create Export → скачайте ZIP
4. Загрузите в Jippy

---

## 5. Processing Pipeline (сервер)

```
                    LIGHT MODE                      FULL MODE
                        │                               │
                        ▼                               ▼
                  Raw text paste                   ZIP file upload
                        │                               │
                        │                               ▼
                        │                    Detect Source
                        │                    (ChatGPT / Claude / Gemini)
                        │                    по структуре файлов в ZIP
                        │                               │
                        │                               ▼
                        │                    Parse Conversations
                        │                    нормализация в единый формат:
                        │                    { question, answer, timestamp,
                        │                      source, conversation_title }
                        │                               │
                        ▼                               ▼
              ┌─────────────────────────────────────────────┐
              │         Extract User Profile                │
              │         LLM-powered extraction:             │
              │         profession, interests, style,       │
              │         language, location, preferences     │
              └──────────────────┬──────────────────────────┘
                                 │
                                 ▼
              ┌─────────────────────────────────────────────┐
              │         Save to Permanent Storage           │
              │         — Q&A pairs (Full) or raw text (L)  │
              │         — User profile (both modes)         │
              │         — Generate embeddings               │
              │         — All raw data stored verbatim      │
              └──────────────────┬──────────────────────────┘
                                 │
                                 ▼
              ┌─────────────────────────────────────────────┐
              │         Add to RAG Buffer                   │
              │         (before first level-up:             │
              │          all imported data = active RAG)    │
              └──────────────────┬──────────────────────────┘
                                 │
                                 ▼
              ┌─────────────────────────────────────────────┐
              │         Delete ZIP from server              │
              │         (keep only parsed data)             │
              └─────────────────────────────────────────────┘
```

### Нормализованный формат Q&A пары
```json
{
  "id": "sha256-hash",
  "question": "Как настроить Google Analytics?",
  "answer": "Для настройки GA нужно...",
  "timestamp": "2025-06-15T14:30:00Z",
  "source": "chatgpt",
  "conversation_title": "Настройка аналитики",
  "topic_keywords": ["analytics", "google", "marketing"],
  "embedding": [0.012, -0.034, ...],
  "raw_question": "...оригинал без обработки...",
  "raw_answer": "...оригинал без обработки..."
}
```

Ключевое: `raw_question` и `raw_answer` хранят оригинал VERBATIM. `question` и `answer` — нормализованные версии для поиска и тренировки. Оригинал никогда не удаляется.

---

## 6. UI/UX Flow — Онбординг

### Экран 1: Выбор режима

```
┌─────────────────────────────────┐
│                                 │
│   Сделай Jippy своим            │
│                                 │
│   ┌─────────────────────────┐   │
│   │  30 сек                 │   │
│   │  Быстрый старт          │   │
│   │  Скопируй память из     │   │
│   │  ChatGPT одним текстом  │   │
│   └─────────────────────────┘   │
│                                 │
│   ┌─────────────────────────┐   │
│   │  5-10 мин               │   │
│   │  Полный импорт           │   │
│   │  Загрузи всю историю    │   │
│   │  из ChatGPT/Claude/     │   │
│   │  Gemini                  │   │
│   └─────────────────────────┘   │
│                                 │
│         Начать с нуля →         │
│                                 │
└─────────────────────────────────┘
```

### Экран 2 (Full): Выбор источника

```
┌─────────────────────────────────┐
│                                 │
│   Откуда импортировать?         │
│                                 │
│   ┌─────────────────────────┐   │
│   │  ChatGPT                │   │
│   │  ZIP-экспорт     [?]    │   │
│   └─────────────────────────┘   │
│                                 │
│   ┌─────────────────────────┐   │
│   │  Claude                 │   │
│   │  ZIP-экспорт     [?]    │   │
│   └─────────────────────────┘   │
│                                 │
│   ┌─────────────────────────┐   │
│   │  Google Gemini           │   │
│   │  ZIP-экспорт     [?]    │   │
│   └─────────────────────────┘   │
│                                 │
│   [?] = показывает инструкцию  │
│                                 │
│   ┌─────────────────────────┐   │
│   │   Загрузить ZIP →       │   │
│   └─────────────────────────┘   │
│                                 │
│   Нет ZIP под рукой?            │
│   Загрузите позже в Настройках  │
│                                 │
└─────────────────────────────────┘
```

### Экран 3: Прогресс (Full mode)

```
┌─────────────────────────────────┐
│                                 │
│   Jippy изучает твою историю... │
│                                 │
│   ████████████░░░ 78%           │
│                                 │
│   Найдено диалогов: 847         │
│   Извлечено знаний: 2,341       │
│   Профиль: определён            │
│                                 │
│   Темы:                         │
│   Наука (312)                   │
│   Код (256)                     │
│   Маркетинг (189)               │
│   ...                           │
│                                 │
└─────────────────────────────────┘
```

### Экран 4: Готово (оба режима)

```
┌─────────────────────────────────┐
│                                 │
│   Jippy тебя знает!             │
│                                 │
│   Light: "Определён профиль"    │
│   Full:  "847 диалогов →        │
│           2,341 знание"         │
│                                 │
│   Jippy узнал что ты:           │
│   - Маркетолог, 5+ лет          │
│   - Работаешь с данными          │
│   - Предпочитаешь русский       │
│   - Любишь детальные ответы     │
│                                 │
│   ┌─────────────────────────┐   │
│   │   Начать общение →      │   │
│   └─────────────────────────┘   │
│                                 │
└─────────────────────────────────┘
```

---

## 7. Требования к мобильному приложению

### Приложение ДОЛЖНО уметь:

1. **Принимать ZIP-файлы:**
   - iOS: `UIDocumentPickerViewController` (Files app, iCloud, Downloads)
   - Android: `Intent.ACTION_OPEN_DOCUMENT` с MIME type `application/zip`
   - Поддержка `.dms` расширения (Claude экспорт)

2. **Share Extension (iOS) / Share Target (Android):**
   - Принимать ZIP через "Поделиться" из Mail, Files, Safari
   - Intent filter: `application/zip`, `application/x-zip-compressed`

3. **Загрузка на сервер:**
   - Multipart upload ZIP на `/api/jippy/import/upload`
   - Progress callback для прогресс-бара
   - Resume при обрыве сети (для больших ZIP >100MB)

4. **Текстовое поле (Light mode):**
   - Multiline input, paste support
   - POST `/api/jippy/import/text`

5. **Импорт из Настроек (не только онбординг):**
   - Доступен повторно в Profile → Settings → Import
   - Поддержка множественного импорта (ChatGPT + Claude)

---

## 8. API Endpoints (серверная часть)

```
POST /api/jippy/import/text
  Body: { text: string }
  Response: { profile: UserProfile, items_count: number }
  Время: <5 секунд

POST /api/jippy/import/upload
  Body: multipart/form-data (ZIP file)
  Response: { job_id: string }
  Начинает фоновую обработку

GET /api/jippy/import/status/:job_id
  Response: { 
    status: "processing" | "done" | "error",
    progress: 0.78,
    conversations_found: 847,
    qa_pairs_extracted: 2341,
    profile: UserProfile | null,
    topics: [{ name: "Наука", count: 312 }, ...]
  }
  Polling каждые 2 секунды во время обработки

GET /api/jippy/import/history
  Response: [{ source: "chatgpt", date: "...", items: 2341 }, ...]
  Список предыдущих импортов
```

---

## 9. Технические ограничения

- **Размер ZIP:** до 1GB. Парсинг на сервере, не на телефоне.
- **Дубликаты:** дедупликация по `sha256(question + answer + timestamp)`.
- **Приватность:** ZIP удаляется после парсинга. Extracted data шифруется в user storage.
- **Формат Gemini нестабилен:** нужен fallback-парсер + error handling.
- **Profile extraction:** LLM-powered (backbone модель), батч по 50 диалогов.

## 10. Метрики успеха

| Метрика | Target |
|---------|--------|
| % новых юзеров которые импортируют (Light или Full) | >40% |
| % выбравших Light mode | >60% (барьер ниже) |
| Retention D7 с импортом vs без | +20% |
| Среднее кол-во знаний после Full import | >500 |
| Время обработки ZIP <500MB | <2 мин |

## 11. Roadmap

| Этап | Scope | Срок |
|------|-------|------|
| **MVP** | Light mode (text paste) + ChatGPT ZIP | 2 недели |
| **V1** | + Claude ZIP + Gemini ZIP + Share Extension | +1 неделя |
| **V2** | + Web upload (jippy.ai/import) + повторный импорт | +1 неделя |
| **V3** | + Attachment parsing (images, files from chats) | +2 недели |
