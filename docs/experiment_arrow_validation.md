# Experiment: Arrow Routing + LoRA Composition Validation

**Дата создания:** 2026-04-21
**Статус:** ready-to-execute
**Owner:** TBD (ресерчер)
**Связано с:** `project_model_marketplace.md`, Era 6 federated knowledge network

---

## Цель

Валидировать что Arrow находит правильные «специалисты» по compositional query на реальных Jippy данных (1313 логотипов с категориями), и что композиция top-K даёт ответ ≈ oracle.

**Если эксперимент проходит** → Era 6 (federated knowledge network) переходит из гипотезы в план.

**Если проваливается** → нужны другие routing/composition подходы (см. блокеры внизу).

---

## Flavor A — Text routing (1-2 дня)

### Setup

- **Source:** 1313 логотипов Jippy с тегами категорий
- **Группировка:** категории с ≥30 примеров (ожидаемо 8-15 категорий)
- **VLM captions:** уже есть (Qwen2.5-VL-7B на Vadim)
- **Embedding model:** `multilingual-e5-small` (118MB)
- **Capability vector per category** = mean embedding всех VLM captions этой категории
- **Backbone:** Kimi K2.5 (через API)

### Test queries (20 шт)

| Тип | Кол-во | Примеры |
|-----|--------|---------|
| Single-match (sanity) | 5 | "heart logo", "cat logo" |
| Obvious compositional | 5 | "japanese heart logo", "minimalist cat logo" |
| **Hard compositional** (защита от риска "CLIP сам матчит") | 5 | "логотип для здорового питания", "бренд для elderly tech", "logo for sustainable fashion" |
| Out-of-distribution | 5 | "квантовая физика", "stock ticker" — должно вернуть low-confidence |

### Conditions

- **B:** Kimi без hints (baseline)
- **A:** Kimi + hints из Arrow top-2 categories
- **O:** Kimi + hints из manually curated categories (oracle)
- **W:** Kimi + hints из random unrelated categories (sanity check, должен быть хуже B)

### Метрики

- **Routing:** top-2 recall на compositional, top-1 accuracy на single-match
- **Composition:** human rating 1-5 (matches query / aesthetic), 20 queries × 4 conditions = 80 outputs
- **Score gap:** cosine score top-1 vs top-3 (отделимость классов)

### Success criteria

- Top-2 recall ≥ 75% на compositional queries
- A condition ≥ 85% от O по human rating
- W condition заметно хуже B (sanity)

### Deliverable

- Jupyter notebook `flavor_a_routing.ipynb`
- CSV с raw scores
- 1-страничный отчёт

---

## Flavor B — Image gen с LoRA композицией (3-5 дней)

**ТОЛЬКО если Flavor A прошёл.**

### Setup

- **Base:** FLUX.2-klein (как в проде)
- **Per category:** micro-LoRA r=4, ~50 примеров, ~30 мин train на RTX 3090
- 8-12 категорий → 8-12 micro-LoRAs (~2MB each)
- **Capability vectors:** SVD из A/B матриц LoRA (Arrow extraction)

### Test queries

Те же 20 что в Flavor A (где это image-gen применимо, ~15 запросов)

### LoRA composition strategies (тестируем все)

| Стратегия | Описание |
|-----------|----------|
| **C1: naive sum** | `alpha_a=1.0, alpha_b=1.0` |
| **C2: weighted** | `alpha_a=0.5, alpha_b=0.5` |
| **C3: routed weights** | `alpha = softmax(cosine scores)` |
| **C4: sequential** | LoRA A для denoising step 1-N/2, LoRA B для N/2-N |

### Conditions

- **B:** FLUX без LoRA (baseline)
- **A1-A4:** FLUX + Arrow-routed top-2 LoRA по стратегиям C1-C4
- **O:** FLUX + LoRA натренированный specifically на пересечении (если найдём ≥30 примеров) или manually curated hint
- **S1:** только top-1 LoRA (single specialist)

### Метрики

- **VLM auto-eval:** Qwen2.5-VL-7B оценивает каждое изображение по 2 yes/no критериям из query (например "is heart" / "japanese aesthetic")
- **Human A/B:** 15 queries × 7 conditions = 105 изображений, попарное сравнение

### Success criteria

- Лучшая стратегия из C1-C4 ≥ 80% от Oracle по VLM eval
- Composition (best of A1-A4) ≥ S1 (т.е. композиция лучше single specialist'а на compositional queries)

### Deliverable

- Скрипты тренинга
- Grid сгенерированных изображений
- CSV метрик
- 1-страничный отчёт

---

## Ресурсы

| Ресурс | Flavor A | Flavor B |
|--------|----------|----------|
| GPU | — (CPU достаточно) | RTX 3090, ~24h |
| API | Kimi ~$5 | — |
| Время человека | 1-2 дня | 3-5 дней |
| **Total cost** | **~$5** | **~$50** |

---

## Что блокирует Era 6 если эксперимент провалится

| Условие | Интерпретация |
|---------|---------------|
| Top-2 recall < 60% на compositional | Arrow недостаточно, нужен **learned router** (другая архитектура) |
| A < 70% от O | Composition сложнее чем hint injection, нужен **retrieval-augmented fine-tuning** |
| Все стратегии C1-C4 < S1 | Multi-LoRA композиция в FLUX в принципе не работает, нужен **другой подход** (например IP-Adapter style controlled generation) |

---

## Связь с большой архитектурой

Каждая category в этом эксперименте = прокси для одного Jippy с узким opinion в federated network.

- 8 categories = 8 «пользователей» в federation
- Compositional query = real-world запрос требующий expertise нескольких людей
- Top-2 routing = «коммутатор находит 2 правильных Jippy»
- Composition = «их hints мерджатся в финальный ответ»

**Если работает на 8 категориях — масштабируется на тысячи** (с иерархическим routing). Это первая реальная валидация Membrane Principle + Hierarchical Arrow на нашем стеке и реальных данных.

---

## Контекст для исполнителя

- **Phase 0 (precedent):** Arrow на 2 LoRA (Python vs Math) дал 95% balanced accuracy + 97% от oracle in-domain. Этот эксперимент проверяет масштабируется ли подход на 8-15 близких categories с compositional queries.
- **Phase 1 (precedent):** `both_alpha=1` композиция теряла к single matching LoRA (-4.6pp python, -7.7pp math). Поэтому в Flavor B тестируем 4 разные стратегии композиции, а не только naive sum.
- **Existing infra:** VLM captioner работает на Vadim, FLUX.2-klein с jppy-logo LoRA в проде, RTX 3090 доступен на Vast.
