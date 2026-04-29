# GPU Model Evolution Plan

## Принцип эволюции

Модель растёт как дерево:
- **Ствол** (attention layers) = "как думать" — переносится между версиями, знания накапливаются
- **Ветки** (experts) = "что знать" — добавляются/расширяются при каждом апгрейде
- **Алфавит** (embedding) = токенайзер — пересоздаётся при смене, учится быстро

Каждая версия строится на предыдущей через transfer learning. Ничего не теряется.

---

## V1 (сейчас) — Proof of Concept

**Цель:** доказать что архитектура работает, запустить клавиатуру, собрать первых юзеров.

| Параметр | Значение |
|---|---|
| Params total | 2.59B |
| Params active | ~479M (top-4 of 32 experts) |
| d_model | 1024 |
| Layers | 12 (6 dense + 6 expert) |
| Experts | 32, top-4 |
| Tokenizer | GPT-2 (50257 vocab) |
| Batch size | 16 (ограничен VRAM) |
| Training | 100K steps, ~4 дня, 2× L20 |
| Data | 70% RU + 30% EN, 13B tokens |
| Tokens seen | ~3.2B (25% датасета) |

**Проблемы V1:**
- GPT-2 токенайзер ломает русский (2-3 tokens на кириллическую букву)
- Batch size 16 слишком мал для 2.59B модели
- 32 experts = optimizer жрёт VRAM → нет места для большего batch
- Результат: английский средний, русский плохой

**Что V1 даёт для V2:**
- Обученные attention layers (12 слоёв) = "как думать" → переносим
- Отлаженные скрипты (train, checkpoint, monitor, export)
- Работающая клавиатура (iOS) с translation, themes, signals
- Работающая инфраструктура (Hetzner → Turkey tunnel, Caddy, endpoints)
- PoC distributed inference на gpu.social

---

## V2 — Production Keyboard

**Цель:** клавиатура которой реально пользуются. Русский на четвёрку, английский на тройку.

| Параметр | Значение |
|---|---|
| Params total | ~800M |
| Params active | ~300M (top-2 of 8 experts) |
| d_model | 1024 (тот же → transfer attention) |
| Layers | 12 (тот же → transfer напрямую) |
| Experts | 8, top-2 |
| Tokenizer | SentencePiece (32K vocab, RU+EN) |
| Batch size | 64-128 (8 experts = меньше VRAM на optimizer) |
| Training | 100K steps, ~4 дня, 2× L20 |
| Data | 90% RU + 10% EN |
| Tokens seen | ~13-26B (100%+ датасета, 1-2 прохода) |

**Что меняется:**
1. **SentencePiece** → "Привет" = 1 token вместо 6-12 → русский x3-5 лучше
2. **8 experts** вместо 32 → optimizer меньше → batch 64-128 → видит ВСЕ данные
3. **Каждый expert крупнее** → лучше качество при тех же ресурсах
4. **90% русский** → фокус на первых юзерах
5. **Transfer attention из V1** → экономим ~30-40% времени обучения

**Для телефона:**
- 8 experts = каждый телефон хранит 1 expert (~50MB)
- Tier 1 (слабый): shared + 2 experts = ~150MB
- Tier 2 (средний): shared + 4 experts = ~250MB
- Tier 3 (мощный): shared + 8 experts = ~450MB (полная модель)

**Как переносим из V1:**
```
V1 attention (12 layers, d=1024) → V2 attention (12 layers, d=1024)
                                    ✅ Копируем напрямую (размер тот же)

V1 experts (32 × small)          → V2 experts (8 × large)
                                    ❌ Не переносим (другой размер)
                                    Учим с нуля, но attention ускоряет в 3-5x

V1 embedding (GPT-2, 50257)      → V2 embedding (SentencePiece, 32K)
                                    ❌ Другой vocab, создаём новый
                                    Учится быстро (1-2 дня)
```

**Timeline:** ~1 неделя после завершения V1
- День 1: обучить SentencePiece токенайзер
- День 2: настроить training с transfer attention
- Дни 3-6: обучение 100K steps
- День 7: экспорт, деплой, подключить к клавиатуре

---

## V3 (реальный) — Fusion + Knowledge Distillation

**Цель:** впитать знания из топовых моделей (Gemma, Qwen, DeepSeek) в маленькие треки через дистилляцию.

| Параметр | Значение |
|---|---|
| Архитектура | PT-MoE V3: 10 треков, TokenDependentMerge |
| d_model | 1024 |
| Треки 0-4 | Базовые (обучены на наших данных) |
| Треки 5-7 | Дистиллированные из Gemma/Qwen/DeepSeek |
| Треки 8-9 | Расширенные базовые |
| Tokenizer | SentencePiece (32K vocab) |
| Training | Vast.ai A100 80GB |
| Data | v3_train.jsonl (366K примеров, RU+EN) |

**Ключевой сдвиг (апрель 2026):** Отказались от "3 замороженных слоя + линейные проекции" (proj_in/proj_out не обучались). Перешли на **полную дистилляцию**: загружаем всю модель-учителя в INT4, собираем (embedding → final_hidden) пары, обучаем маленького студента (3 слоя, ~50M params) воспроизводить трансформацию учителя.

**Результаты дистилляции Gemma 4 31B:**
- Cosine 0.81 (10K steps), растёт с количеством шагов
- 100K steps в процессе

**Идея персональных треков:** Каждый трек = узкий специалист (Python, русская литература, математика). Пользователь может доучить свой трек (~50MB) на своих данных.

**Было запланировано (MPOE, ReMoE, Tucker)** — отложено, т.к. Fusion+Distillation показал более перспективный путь.

**Главные изменения в V3:**

**1. MPOE — Matrix Product Operator Experts**
```
V2 эксперты: 8 × FFN(1024→4096→1024) = 8 × 50M = 400M params
V3 MPOE:     shared_core + 64 per_expert_delta
             shared_core: FFN(1024→1024→1024) × 6 слоёв = 12.6M
             delta per expert: rank-64 matrices × 6 слоёв = 1.6M/expert
             64 experts: 64 × 1.6M + 12.6M = ~115M total

Сжатие: 400M → 115M = 3.5× (при том же числе экспертов)
На телефоне: shared_core (6.3MB) + 1 expert delta (0.8MB) = <10MB expert part
```

**2. ReMoE routing**
```
V2: softmax(scores) → top-k mask (дискретно, не дифференцируемо)
V3: ReLU(scores) / L1_norm(scores) (непрерывно, адаптивная спарсивность)

- Токен сам решает сколько экспертов активировать
- Лучше validation loss чем top-k при n_experts=64
- Drop-in замена, без изменения остальной архитектуры
```

**3. Early Exit (активируем то что уже есть)**
```
EarlyExit heads уже в gpu/model.py, нужно только включить:

⚡ Auto Mode: модель выходит на любом слое когда confidence > 0.85
   - keyboard autocomplete: обычно слой 3-4 (экономим 8 слоёв)
   - сложный вопрос: все 16 слоёв

Обучение: loss на каждом exit head → модель учится быть полезной в любой точке
```

**4. +4 слоя TN-сжатые**
```
V2: 12 слоёв → V3: 16 слоёв
Лишние 4 слоя: Tucker/TT декомпозиция весов
Сжатие слоя: 3-5× → практически "бесплатные" 4 слоя по вычислениям
Глубина: +33% = лучше рассуждения и контекст
```

**Как переносим из V2:**
```
V2 attention (d=1024, 12 layers) → V3 attention (d=1024, 16 layers)
                                    ✅ Первые 12 — копируем напрямую
                                    4 новых — инициализируем из соседних V2

V2 experts (8 × large FFN)       → V3 MPOE (64 × tiny delta + shared_core)
                                    ❌ Другая факторизация, учим с нуля
                                    Но attention из V2 ускоряет ~40%

V2 routing (top-k softmax)       → V3 ReMoE (ReLU adaptive)
                                    Замена только router head
```

**На телефоне (V3):**
```
Shared: ~100MB (attention 16 layers + embedding + LM head)
Expert delta: ~1MB (rank-64, 6 layers)
Shared core: ~6MB (один для всех)
Итого: ~107MB — well under 200MB target!
→ Можно хранить 2-3 expert deltas = 108-110MB
```

**Новое в V3:**
- On-device inference (мини-модель на телефоне, Early Exit)
- Federated learning (телефоны шлют градиенты, не текст)
- Prediction strip с фразами
- Персонализация: per-user fine-tune expert delta (~1.6M params, очень быстро)

---

## V4 — Distributed Intelligence

**Цель:** модель полностью на телефонах, API для всех, телефоны зарабатывают за compute.

| Параметр | Значение |
|---|---|
| Params total | ~10B+ |
| Params active | ~1B (top-2 of 32 experts) |
| d_model | 4096 |
| Layers | 24 |
| Experts | 32, top-2 |
| Training | distributed (телефоны + GPU) |
| Network | **QUIC** (через iroh) вместо WebSocket |
| Discovery | **Nostr** Kind 31990 вместо signaling server |
| Demand | **Gossip** protocol, TTL decay, auto standby-promote |

**Как переносим из V3:**
```
V3 (12 layers, d=2048) → V4 (24 layers, d=4096)
                          12 старых layers → pad → копируем
                          12 новых layers → инициализируем из старых
                          d: 2048 → pad → 4096
```

**Новое в V4:**
- Сервер = только координатор (не делает inference)
- Телефоны считают inference и обучение
- API: запросы идут на телефоны через координатор
- Монетизация: телефоны зарабатывают за compute
- Модель на уровне GPT-2 XL / начального GPT-3

---

## Сводная таблица эволюции

```
         Params   d_model  Experts     FFN   Routing    Layers  Per-phone  Русский  Английский  Где живёт
V1       2.59B    1024     32/top4    4096   softmax+k   12     ~200MB     ❌ 2/5   ⚠️ 3/5      Сервер
V2       800M     1024      8/top2    4096   softmax+k   12     ~150MB     ✅ 4/5   ⚠️ 3/5      Сервер → телефон
V3       ~500M    1024     64/MPOE   1024*  ReMoE       16     ~110MB     ✅ 4.5/5 ✅ 4/5      Телефон (hybrid)
V4       10B+     4096     32/top2    4096   ReMoE+MPOE  24     ~400MB     ✅ 5/5   ✅ 5/5      Телефоны (P2P)
```
*MPOE: expert delta rank-64, shared_core=1024

## Принцип: Transfer Chain

```
V1 attention ──copy──→ V2 attention ──pad──→ V3 attention ──pad──→ V4 attention
     │                      │                     │                     │
  "базовое              "хороший              "продвинутое         "глубокое
   понимание             русский"               мышление"           рассуждение"
   языка"
```

Знания НАКАПЛИВАЮТСЯ. Каждая версия стартует не с нуля а с того места где остановилась предыдущая. Как ребёнок → школьник → студент → профессор.

## Автоматическая эволюция (между версиями)

Между ручными апгрейдами модель улучшается сама:
- Юзеры печатают → signals (predicted vs typed)
- Сервер/телефоны считают градиенты → median aggregation
- Модель обновляется → predictions точнее → больше юзеров

Каждый апгрейд (V1→V2→V3→V4) = "перестройка мозга". Между апгрейдами = "ежедневная практика".

## Продукты (Tools) — каждый учит модель

Каждый продукт = инструмент для людей + генератор training data для модели.

### Tool 1: GPU Keyboard (V1-V2)
**Что делает:** клавиатура с AI predictions, translation, скины.
**Как учит модель:** каждое набранное слово = predicted vs typed signal. Персонализация стиля.
**Training data:** next-word prediction pairs, acceptance/rejection signals.
**Языки:** RU, EN.
**Платформы:** iOS (готово), Android (следующий).

### Tool 2: Babel Chat (V2-V3)
**Что делает:** мессенджер без языковых границ. Пишешь на своём языке — все читают на своём. Автоперевод в реальном времени.
**Как учит модель:** каждое сообщение = translation pair (исходный язык → целевой язык). Юзеры поправляют переводы → supervised signal. Миллионы пар в день.
**Training data:** parallel text pairs на десятках языков, human corrections.
**Killer feature:** не нужно учить язык чтобы общаться с кем угодно.
**Почему это идеально для нас:**
- Translation = самый ценный тип данных для языковой модели
- Каждое исправление перевода юзером = бесплатный human label
- Больше языков → модель учит структуру ВСЕХ языков → лучше на каждом
- Viral loop: приглашаешь друга из другой страны → он тоже приглашает → рост
**Реализация:** WebSocket чат на gpu.social + мобильное приложение. Translation через наш сервер (сейчас OPUS-MT, потом наша модель).

### Tool 3: GPU API (V3-V4)
**Что делает:** открытый API для inference. Любой разработчик может подключиться.
**Как учит модель:** API запросы = diverse prompts, расширяют знания модели.
**Training data:** prompt-completion pairs из реального использования.
**Монетизация:** freemium — бесплатно до N запросов/день, платно выше.

### Tool 4: Community Tools (V3+)
**Что делают:** маркетплейс AI-тулов от community (vibe coders).
**Примеры:**
- Speech-to-text → голосовые данные для модели
- Image description → multimodal data
- Code assistant → code training pairs
- Document summarizer → compression/understanding data
**Как учит модель:** каждый инструмент генерирует специализированные данные.
**Принцип:** коммьюнити строит тулы → тулы генерят данные → модель умнеет → тулы работают лучше → маховик.

### Как тулы связаны с эволюцией модели

```
V1: Keyboard (EN/RU)          → next-word signals     → модель учит языки
V2: Keyboard + Babel Chat     → translation pairs      → модель учит переводы
V3: + API + Community Tools   → diverse data           → модель учит всё
V4: Всё distributed           → телефоны учат сами     → модель растёт бесконечно
```

Каждый продукт = насос данных для модели. Больше продуктов → больше типов данных → модель умнее → продукты лучше.

---

## Babel Chat — подробнее

### MVP (2-3 недели)
- Веб: gpu.social/chat — заходишь, создаёшь комнату, кидаешь ссылку
- Мобилка: отдельная апка или вкладка в GPU Keyboard app
- WebSocket чат + автоперевод каждого сообщения
- Язык определяется автоматически
- Каждый видит все сообщения на СВОЁМ языке
- Оригинал показывается мелким текстом под переводом (чтобы проверить)

### UX
```
┌─────────────────────────────────┐
│ 🌐 Babel Chat — Room "devs"    │
├─────────────────────────────────┤
│                                 │
│ 🇷🇺 Юка:                       │
│ Привет, как продвигается?       │
│                                 │
│ 🇺🇸 Mike:                       │
│ Всё хорошо, закончил API       │
│ ᵉⁿ "Going well, finished API"  │
│                                 │
│ 🇯🇵 Yuki:                       │
│ Круто! Я тоже почти готова     │
│ ʲᵃ "すごい！私もほぼ完了"         │
│                                 │
├─────────────────────────────────┤
│ [📎] Напишите сообщение...  [→] │
└─────────────────────────────────┘
```

Каждый пишет на своём языке. Каждый читает на своём языке. Оригинал виден мелко.

### Как учит модель
- Сообщение на RU → автоперевод на EN, JA, ES...  = N translation pairs
- Юзер видит плохой перевод → тапает "исправить" → human-corrected pair (золотые данные!)
- 100 юзеров × 50 сообщений/день × 3 языка = 15,000 translation pairs/день
- Через месяц: 450,000 пар → fine-tune translation модели → перевод лучше → больше юзеров

---

## Stage 4: Ambient Intelligence (8-12 мес)

**Цель:** модель знает что тебе нужно ДО того как ты спросил

### Модель V4
- 8B+, d_model=4096, 32 experts
- Instruction tuning (модель следует инструкциям)
- RLHF от юзеров (thumbs up/down в клавиатуре и чате)

### Контекстные сигналы (с явного разрешения юзера!)
- Время суток → предзагрузка experts (утро = деловые, вечер = casual)
- Геолокация → язык и тематика (ресторан = еда, офис = работа)
- Календарь → "Meeting with John at 3pm" → предзагрузка английских experts
- Ambient audio (opt-in) → определение обстановки (шумно/тихо/музыка)

### Продукты
- **Smart Compose**: overlay поверх ЛЮБОГО приложения (как Grammarly, но AI)
- **Voice Assistant**: голосовой ввод + ответы на базе нашей модели

### Цель
- 100,000+ юзеров
- Сеть самодостаточна (inference полностью на телефонах)
- Модель предсказывает потребности юзера

---

## Stage 5: Smart Agent — "Сын Антона" (12-18 мес)

**Цель:** модель = полноценный агент с инструментами

### Модель V5
- 20B+, instruction-tuned, RLHF, tool-use
- Модель САМА выбирает какой инструмент использовать (MCP-style)

### Tool-Use (MCP-style)
Модель получает каталог доступных tools и сама решает когда какой вызвать:
```
Юзер: "сколько $150 в рублях?"
Модель: [tool: currency_converter, args: {from: USD, to: RUB, amount: 150}]
→ "примерно 13,500₽"
```
10+ встроенных: перевод, поиск, калькулятор, погода, календарь, заметки,
напоминания, генерация текста, саммари, grammar fix.

### MCP Tool Marketplace
- Разработчики и юзеры создают tools (MCP серверы)
- Мы проверяем и публикуем (как App Store review)
- Модель учится использовать новые tools от юзеров
- Юзеры делятся tools друг с другом

### Архитектура
- 95% distributed (inference на телефонах)
- 5% centralized (обновления модели, проверка tools, мониторинг health сети)
- Полная decentralization = open research question, решаем когда готовы

### Персонализация через Evolutionary Experts
- Каждый юзер = эксперт в своей теме
- Модель на ТВОЁМ телефоне знает ТЕБЯ лучше любого другого AI
- Для задач вне твоей экспертизы — дёргает experts других юзеров в сети
- Правило: не убивать expert если замены в сети нет (знания не теряются)

### Цель
- 1,000,000+ юзеров
- Marketplace с 100+ tools
- Модель = personal AI assistant
- **Не пытаемся быть ChatGPT. Мы — лучшая клавиатура в мире которая понимает тебя лучше чем ты сам.**

---

## Optimization Roadmap (across all stages)

### Speed (tok/s между телефонами)
1. **Pipeline Parallelism** → throughput x4 (Stage 2)
2. **Speculative Decoding** с готовой мини-моделью (rugpt3small/distilgpt2) → x3-5 (Stage 2) — **✅ ПОДТВЕРЖДЕНО mesh-llm**: +38% throughput на коде, 75% acceptance rate. ([docs.anarchai.org](https://docs.anarchai.org))
3. **Geographic Clustering через P2P WiFi** → телефоны в одном помещении общаются НАПРЯМУЮ без интернета (Stage 3)
   - iOS: `MultipeerConnectivity` framework (тот же протокол что AirDrop)
   - Android: `Nearby Connections API` (Google)
   - Latency <1ms вместо 40-100ms через сервер
   - Работает без интернета (самолёт, метро, подвал)
   - Сценарий: 5 человек в офисе = полная модель локально, быстрее облака
4. **Expert Caching** на edge nodes / WiFi роутерах (Stage 3)
5. **Predictive Pre-computation** → модель считает следующий токен заранее → x2-3 (Stage 3)

### Intelligence (качество модели)
1. **SentencePiece tokenizer** → русский x3-5 лучше (Stage 1→2)
2. **Sleep Learning** → ночная тренировка на зарядке + бесплатный compute (Stage 2)
3. **Crowd Wisdom Routing** → "юзеры похожие на тебя активировали experts [3,7,15]" (Stage 3)
4. **Evolutionary Experts** → естественный отбор: сильные experts размножаются, слабые эволюционируют. Каждый юзер = эксперт в своей теме. Не убивать без замены. (Stage 3)
5. **Mixture of Depths** → лёгкие токены пропускают слои (entropy-based), сложные получают полный compute. Средняя скорость x2 (Stage 3)
6. **Hierarchical Experts** → domain router (Tech|Food|Emotions) → specific expert. Routing x3 быстрее (Stage 3)
7. **RLHF от юзеров** → thumbs up/down → модель учится из фидбека (Stage 4)

### Efficiency (меньше ресурсов → лучше output)
1. **INT4 Quantization** → модель в 8x меньше, 800M = 400MB на телефоне (Stage 2)
2. **Flash Attention** → O(n) память → context 2048 на телефоне (Stage 2)
3. **KV-Cache Sharing** → общий кэш на чат-комнату → x100 экономия compute (Stage 2)
4. **Ambient pre-loading** → experts подгружаются ДО начала печати → zero-latency (Stage 4)

---

## Реалистичность

| Stage | Timeline | Вероятность | Самое сложное |
|-------|----------|-------------|---------------|
| 1 Foundation | сейчас → 2 мес | 95% | Уже почти сделано |
| 2 Smart Network | 2-4 мес | 85% | Pipeline parallelism на телефонах |
| 3 Distributed Intelligence | 4-8 мес | 70% | Geographic clustering + evolutionary experts |
| 4 Ambient Intelligence | 8-12 мес | 50% | Глубокая интеграция с OS, разрешения юзеров |
| 5 Smart Agent | 12-18 мес | 40% | 20B distributed + tool-use quality + MCP marketplace |

Stages 1-3 полностью реальны с нашими ресурсами.
Stage 4 требует глубокой работы с OS permissions.
Stage 5 = frontier research, но каждый элемент по отдельности решаем.

---

## Что нужно для каждого шага

| Шаг | GPU | Время | Зависимость | Продукт |
|-----|-----|-------|-------------|---------|
| V1→V2 | 2× L20 (есть) | ~1 неделя | V1 завершён | Keyboard RU |
| Babel MVP | — (OPUS-MT на CPU) | ~2-3 недели | Translation server | Babel Chat |
| V2→V3 | 2× A100 или 4× L20 | ~2 недели | 1000+ юзеров | Keyboard + Babel + API |
| V3→V4 | 8× A100 или distributed | ~1 месяц | 10000+ юзеров | Full ecosystem |
| V4→V5 | Distributed (телефоны) | ongoing | 100000+ юзеров | Agent + Marketplace |
