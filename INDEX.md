# INDEX — навигация по репозиторию

**Цель этого файла:** одна точка входа. Когда не знаешь куда положить новую инфу или где её искать — смотри сюда.

**Правило:** новая инфа идёт ТОЛЬКО в один из 3 канонических файлов. Всё остальное — архив.

---

## 🟢 Канонические файлы (живые, редактируемые)

### 1. `docs/jippy_architecture_final.md` — **СПЕКА продукта Jippy**
Сюда идёт всё про:
- Архитектура Jippy (телефон + сервер)
- Inference / level-up / import flows
- Хранение данных, приватность
- API endpoints
- Tool use на телефоне (раздел 10)
- Sensor pipeline (раздел 11)
- Continuous learning / TTT (раздел 12)
- Любое уточнение продукта или архитектуры

**Если не уверен куда — кладёшь сюда.**

### 2. `FRONTIER_ATTACK.md` — **СТРАТЕГИЯ против frontier moделей**
Сюда идёт всё про:
- Какие архитектурные дыры мы атакуем у OpenAI/Anthropic/Gemini
- Sequencing (что строим первым, вторым, третьим)
- Что НЕ можем атаковать (честные ограничения)
- Маркетинговые обещания (что говорим / не говорим)
- Свои бенчмарки

### 3. `INDEX.md` — **этот файл**
Сюда идёт:
- Навигация и правила работы с файлами
- Когда добавляешь новый канонический файл — обнови этот

---

## 🟡 Активные но узкие (не трогать без причины)

| Файл | Назначение |
|------|-----------|
| `docs/TRAINING_RULES.md` | Правила тренировки моделей в gpu.social marketplace |
| `docs/architecture_gap_loop.md` | Текущая research-петля (post-Phase-1) |
| `docs/experiment_arrow_validation.md` | Подробности эксперимента Arrow routing |
| `docs/experiment_composition_test.md` | Подробности multi-LoRA композиции |
| `docs/feature_spec_hybrid_search.md` | Спецификация hybrid search engine |
| `docs/feature_spec_import_ai_history.md` | Спецификация ZIP-импорта |
| `docs/feature_spec_sonar.md` | Спецификация Sonar feature |
| `docs/pilot_sonar_founder.md` | Пилот Sonar для основателей |
| `docs/IMPLEMENTATION_PLAN.md` | 🆕 Build order для backend MVP. SOT для порядка работы. Реализует §17 Memory Architecture. |
| `docs/JIPPY_BACKEND_USER_TODO.md` | 🆕 P0/P1/P2 todos что нужно от Юки для production (backbone, Sonar key, eval ground-truth, hard rules, whitelist) |

Это **узкие технические спеки**. Редактируем только если меняется именно эта фича. Не дублируем сюда общую инфу.

---

## 🔴 Архив (исторические снимки, НЕ редактировать)

Эти файлы — снимки прошлых решений. Полезны как контекст «как мы пришли к текущему», но **новую инфу сюда не добавляем**.

| Файл | Что было | Заменено |
|------|----------|----------|
| `JIPPY_TZ.md` | Старый MVP TZ с Тамагочи-геймплеем (Apr 9) | `docs/jippy_architecture_final.md` |
| `JIPPY_AI_ARCHITECTURE.md` | Research-перспектива Phase 0/1 (Apr 11) | `docs/jippy_architecture_final.md` + memory |
| `GPU_PLAN.md` | Старый план gpu.social (Apr 2) | `FRONTIER_ATTACK.md` |
| `GPU_EVOLUTION.md` | Эволюция gpu.social видения (Apr 8) | `FRONTIER_ATTACK.md` + memory |
| `SOLUTIONS.md` | Решения проблем (Apr 8) | Раздроблено в спеки |
| `PROBLEMS.md` | Каталог проблем (Apr 2) | Большинство решено или в FRONTIER_ATTACK |
| `COMPETITORS.md` | Анализ конкурентов (Apr 2) | Включено в FRONTIER_ATTACK |
| `UNDERDOGS.md` | Исследование underdog-стратегий (Mar 28) | Дух перешёл в FRONTIER_ATTACK |
| `EXPERIMENTS.md` | История экспериментов (Apr 8) | Текущие — в `docs/experiment_*` |
| `PRODUCT_FLOW.md` | Старый product flow (Apr 8) | `docs/jippy_architecture_final.md` |
| `DEVELOPER_GUIDE.md` | Разработческий гайд (Mar 23) | Устарел |
| `apps.md` | Старые наброски приложений (Mar 26) | Устарел |
| `ataturk_turkish_data_sources.md` | Источники данных для турецкой модели | Узкая research-вспомогалка |
| `georgian_cuisine_data_sources.md` | Источники данных для грузинской модели | Узкая research-вспомогалка |
| `README.md` | Базовое описание репо | Можно обновить, но не приоритет |

**Принцип:** не удаляем (история ценна), но не редактируем. Если нужно сослаться на исторический момент — линкуем оттуда сюда.

---

## ✅ Правила работы

1. **Любая новая инфа про продукт** → `docs/jippy_architecture_final.md`
2. **Любая новая инфа про стратегию** → `FRONTIER_ATTACK.md`
3. **Создаёшь новый файл?** Обнови этот INDEX. Иначе через месяц забудется.
4. **Хочешь дописать в архивный файл?** → подумай, не правильнее ли в один из канонических. В 95% случаев — правильнее.
5. **Memory (`~/.claude/projects/.../memory/MEMORY.md` и связанные)** дублирует key insights отсюда. Поддерживается отдельно через `consolidate-memory` skill.
6. **Будущие идеи (не реализовывать сейчас):**
   - Feature/product идеи → §18 Backlog в `docs/jippy_architecture_final.md`
   - Strategy/attack идеи → Strategic Backlog в `FRONTIER_ATTACK.md`
   - Никаких отдельных файлов для idea-парковки. Жизненный цикл: parked → queued → promoted (в нужный раздел) → rejected.

---

## 📌 Текущее состояние (2026-04-27)

- **Phase 0 Jippy:** ✅ завершено, LoRA воспроизводит Yuka voice + CC engineering voice
- **Phase 1 multi-LoRA:** Arrow routing валидирован 95% balanced accuracy
- **Frontier Attack:** документ создан, 8 структурных дыр размечены
- **Tool use / sensors / TTT:** добавлены в спеку (разделы 10-12)
- **On-device training:** ✅ валидирован end-to-end на M2 (TG → 14.5 MB LoRA → working voice). См. §15 architecture_final, апдейт в FRONTIER_ATTACK Фаза А.2.
- **Conservative weight updates (§12.6):** знакомые из close circle (whitelist 20-50, k≥3, grounded в их (a)) могут менять веса. Незнакомцы — только в RAG, никогда в веса. Quality control = outcome-based utility (RLVR-style), без human votes.
- **Proximity Mesh & LoRA Hunt (§16):** BLE AK-47 v2 protocol (rotating pubkey, nonce, mutual Accept, 60-сек window, freshness decay, adaptive battery). Identity-as-mastery: Pokemon-style теги+уровни+бейджи вместо имени/лица. 5 слоёв (Map/Quest/Meet/Collect/Host). Закрывает дыры #3 и #5 в FRONTIER_ATTACK — структурно невозможно для облачного AI.
- **🆕 Memory Architecture (§17):** **архитектурный фундамент**. 5 слоёв через memory taxonomy (Tulving/Squire): Слой 0 World Knowledge (backbone+web+cache), Слой 1 Native Search (episodic, no copies), Слой 2 Structured RAG (5 подслоёв включая Knowledge Map типа Obsidian с GraphRAG extraction), Слой 3 LoRA (procedural+stylistic, NOT facts), Слой 4 FT (server community + L10 milestone). Routing rules + cheat sheet «что куда». Все остальные разделы ссылаются сюда.
- **🆕 Backend MVP shipped:** все 4 фазы IMPLEMENTATION_PLAN.md реализованы в `~/Desktop/LLM/jippy/backend/` (uncommitted, working). 608K events ingested, SQLCipher encrypted, 28 golden queries, harness routing per §17.8 (heuristic intent classifier), LoRA registry, multi-voice variant A+C. P0/P1 todos в `docs/JIPPY_BACKEND_USER_TODO.md`.
- **🆕 §18 Backlog (product) + Strategic Backlog (FRONTIER_ATTACK):** парковки для будущих идей. Idea 1 в §18 — **Mindful Cart** (food-as-program, дыра #5 grounding). Strategic Idea 1 — **Enterprise track** (org-scale LoRA stack, расширение дыры #1 «Scope Hierarchy»).
- **Hermes agent_loop adoption (Phase 3+):** vendor ~700 LOC из NousResearch/hermes-agent (MIT) для multi-step reasoning. Qwen уже эмитит их формат нативно — zero-friction. См. `IMPLEMENTATION_PLAN.md` Phase 3+ + `JIPPY_BACKEND_USER_TODO.md` §9.
- **🆕 Tiered User Profile (§17.5.1 redesign):** schema эволюционировала — fact/inference/hypothesis tier'ы с confidence + provenance + active testing. Каждое поле живёт в одном из 3 tier'ов, hypotheses генерируются Tier 3 LLM synthesis из cross-field reasoning, harness проактивно тестирует low-confidence hypotheses, decay устаревшие. Источник идеи: AlexShrestha/marble (kg.js + hypothesis-driven cold start). Передаётся разработчику. См. `JIPPY_BACKEND_USER_TODO.md` §10.
- **🚀 Next Sprint pinned (2026-04-29):** **6 items** must ship — S1 Layer 2c NER rebuild, S2 Closeness multi-signal scoring, S3 Profile UI, S4 People UI (kanban drag-and-drop), S5 Knowledge Map UI (force-directed graph), **S6 Harness v2 (parallel fetch + adaptive context)**. Backend ~5-6 дней, Frontend ~7-9 дней. Full breakdown в `IMPLEMENTATION_PLAN.md` Next Sprint + `JIPPY_BACKEND_USER_TODO.md` §11-15.
- **§17.8 routing переписан v2:** убран regex intent classifier и per-intent assembly. Заменено на parallel fetch всех Layer 1+2 + LLM-as-router через rich context. Layer 0 split: 0a parametric default (free), 0b Sonar только при temporal markers. Adaptive chat history compression — 10K tokens budget + summary fallback.
- **Hard rules check** убран из harness routing до Phase Б — спека §17.8 обновлена. Запреты пока через system prompt инструкции.
- **Backbone roadmap зафиксирован:** все LLM-вызовы через Cerebras 70B → Cerebras Paid → Groq fallback до момента когда LoRA готова. Switch criteria в §17.8 «Backbone roadmap».
