# Experiment: Top-K Weighted Composition Test

**Дата создания:** 2026-04-21
**Статус:** ready-to-execute
**Owner:** TBD (ресерчер)
**Связано с:** `project_lora_learnings.md` (TODO-compo, TODO-debias), Era 6 validation
**Precedent:** 19-LoRA Era 6 experiment (2026-04-21) — winner-take-all валидирован, composition НЕ протестирован

---

## Цель

Проверить работает ли **top-K weighted composition** на compositional queries («japanese heart logo» и т.п.). Текущий winner-take-all даёт только один концепт. Нужно протестировать: даёт ли смешение 2-3 LoRA с весами **оба концепта** в результате.

**Если работает** → Era 6 composition разблокирована, можно делать federated knowledge composition между «специалистами» (= между Jippy разных пользователей).
**Если не работает** → нужен другой механизм (например LoRA merge → fine-tune, или IP-Adapter style).

---

## Prerequisites (что уже есть)

- ✅ 19 LoRA натренированы на FLUX.2-klein, rank=4 (`~/Desktop/LLM/backups/lora-era6-19/` + Vadim mirror)
- ✅ Capability vectors извлечены, cos matrix построена
- ✅ E2E test framework работает (`~/Desktop/LLM/step0_inventory/_e2e_19_results/`)
- ✅ VLM eval (Qwen2.5-VL-7B) на Vadim
- ✅ HTML gallery output паттерн (см. `_e2e_19_results/gallery.html`)

---

## Test Queries (REVIEW REQUIRED — пользователь корректирует)

20 запросов, разбиты по типам. **Финальный список зависит от точных 19 категорий.** Скорректировать перед запуском.

### Type A: Compositional 2-concept (12 запросов — основной test)

| # | Query | Expected top-2 LoRAs |
|---|-------|----------------------|
| 1 | japanese heart logo | japanese + heart |
| 2 | japanese minimalist logo | japanese + minimalist |
| 3 | japanese kids brand logo | japanese + kids |
| 4 | minimalist heart icon | minimalist + heart |
| 5 | kids sport team logo | kids + sport |
| 6 | abstract sport emblem | sport + abstract |
| 7 | moon film festival logo | moon + film |
| 8 | japanese moon brand | japanese + moon |
| 9 | minimalist food delivery brand | minimalist + fooddelivery |
| 10 | heart for kids charity | heart + kids |
| 11 | japanese sport team emblem | japanese + sport |
| 12 | retro food delivery service | retro? + fooddelivery |

### Type B: Compositional 3-concept (3 запроса — stress test для C3)

| # | Query | Expected top-3 |
|---|-------|----------------|
| 13 | japanese minimalist heart | japanese + minimalist + heart |
| 14 | kids japanese food brand | kids + japanese + food |
| 15 | minimalist abstract sport logo | minimalist + abstract + sport |

### Type C: Single-concept controls (3 запроса — sanity, ожидаем W не хуже C2)

| # | Query | Expected top-1 |
|---|-------|----------------|
| 16 | heart logo | heart |
| 17 | japanese logo | japanese |
| 18 | minimalist icon | minimalist |

### Type D: Edge cases (2 запроса — robustness)

| # | Query | Note |
|---|-------|------|
| 19 | sushi chef cartoon mascot | unusual phrasing, expects food + something cartoonish |
| 20 | wedding band heart logo | "band" может вытянуть music vs heart, проверка робастности |

**⚠️ User review:** заменить queries которые ссылаются на категории не существующие в реальных 19 LoRA. Указать актуальные имена LoRA для expected top-K.

---

## Conditions (8 на каждый запрос)

| Cond | Описание |
|------|----------|
| **B** | Baseline FLUX без LoRA |
| **W** | Winner-take-all: top-1, weight=1.0 (текущий продакшн baseline) |
| **C2-eq** | Top-2, equal weights `[0.5, 0.5]` |
| **C2-soft** | Top-2, weights = `softmax([gap_top1, gap_top2])` |
| **C2-norm** | Top-2, weights = `[gap_top1, gap_top2] / sum` |
| **C3-soft** | Top-3, weights = `softmax(top3 gaps)` |
| **DB-C2-soft** | **De-biased:** split query на atomic concepts → embed каждый отдельно → average → routing top-2 + softmax weights |
| **O** | Oracle: вручную выбрать «правильные» top-2 LoRA + вручную подобрать weights (ground truth ceiling) |

### Implementation pseudo-code

```python
# Загрузка
pipe = FluxPipeline.from_pretrained("black-forest-labs/FLUX.2-klein-4B", ...)
for lora_name in lora_names:
    pipe.load_lora_weights(f"~/lora-era6-19/{lora_name}", adapter_name=lora_name)

embedder = SentenceTransformer("multilingual-e5-small")  # или Qwen embedding если он использовался для capability vectors
capability_vectors = load_npy("~/lora-era6-19/capability_vectors.npy")  # 19 × dim

def route(query, k=2, debias=False):
    if debias:
        atomic = split_to_atomic(query)  # см. ниже
        embs = [embedder.encode(a) for a in atomic]
        emb = np.mean(embs, axis=0)
    else:
        emb = embedder.encode(query)
    
    scores = cosine_similarity(emb, capability_vectors)
    top_k_idx = np.argsort(scores)[-k:][::-1]
    top_k_scores = scores[top_k_idx]
    return top_k_idx, top_k_scores

def split_to_atomic(query):
    """Простая heuristic: '<adj1> <adj2> ... <noun>' → ['<adj1> <noun>', '<adj2> <noun>']
    Например: 'japanese heart logo' → ['japanese logo', 'heart logo']
    """
    tokens = query.lower().split()
    noun = tokens[-1]  # 'logo' / 'icon' / 'brand'
    modifiers = [t for t in tokens[:-1] if len(t) > 2]
    return [f"{m} {noun}" for m in modifiers]

# Conditions
def generate(query, condition):
    if condition == "B":
        pipe.set_adapters([])
        return pipe(query).images[0]
    
    if condition == "W":
        idx, scores = route(query, k=1)
        if scores[0] < 0.3:  # abstain (как в проде)
            return pipe(query).images[0]
        pipe.set_adapters([lora_names[idx[0]]], adapter_weights=[1.0])
    
    elif condition == "C2-eq":
        idx, scores = route(query, k=2)
        if max(scores) < 0.3: return pipe(query).images[0]  # abstain
        pipe.set_adapters([lora_names[i] for i in idx], adapter_weights=[0.5, 0.5])
    
    elif condition == "C2-soft":
        idx, scores = route(query, k=2)
        if max(scores) < 0.3: return pipe(query).images[0]
        weights = softmax(scores).tolist()
        pipe.set_adapters([lora_names[i] for i in idx], adapter_weights=weights)
    
    elif condition == "C2-norm":
        idx, scores = route(query, k=2)
        if max(scores) < 0.3: return pipe(query).images[0]
        weights = (scores / scores.sum()).tolist()
        pipe.set_adapters([lora_names[i] for i in idx], adapter_weights=weights)
    
    elif condition == "C3-soft":
        idx, scores = route(query, k=3)
        if max(scores) < 0.3: return pipe(query).images[0]
        weights = softmax(scores).tolist()
        pipe.set_adapters([lora_names[i] for i in idx], adapter_weights=weights)
    
    elif condition == "DB-C2-soft":
        idx, scores = route(query, k=2, debias=True)
        if max(scores) < 0.3: return pipe(query).images[0]
        weights = softmax(scores).tolist()
        pipe.set_adapters([lora_names[i] for i in idx], adapter_weights=weights)
    
    elif condition == "O":
        # Manual: для каждого query руками заданы oracle_top2_idx и oracle_weights
        idx, weights = ORACLE_MAPPING[query]
        pipe.set_adapters([lora_names[i] for i in idx], adapter_weights=weights)
    
    return pipe(query).images[0]

# Main loop
for query in test_queries:
    for cond in ["B", "W", "C2-eq", "C2-soft", "C2-norm", "C3-soft", "DB-C2-soft", "O"]:
        img = generate(query, cond)
        save_image(img, f"out/{query_id}/{cond}.png")
```

**Seed:** фиксировать одинаковый seed для всех conditions одного query, чтобы изменения были только за счёт LoRA composition (не diffusion noise).

---

## Eval

### A. VLM auto-eval (Qwen2.5-VL-7B)

Per query, для каждого изображения **2 yes/no вопроса** (по концептам в query):

```
Query: "japanese heart logo"
Concept_1: japanese
Concept_2: heart

VLM prompt (per image):
"Look at this logo image. Answer YES or NO:
Q1: Does this logo show Japanese aesthetic (e.g., enso, kanji, hanko, minimalist asian style)?
Q2: Does this logo contain or resemble a heart shape?

Respond strictly:
Q1: YES/NO
Q2: YES/NO"
```

**Score per image:** `(has_concept_1 + has_concept_2) / 2` — partial credit (0, 0.5, 1).

Для 3-concept (Type B): `(has_1 + has_2 + has_3) / 3`.

### B. Gallery output (КРИТИЧНО — пользователь хочет смотреть сам)

HTML галерея в формате `~/Desktop/LLM/step0_inventory/_e2e_composition_results/gallery.html`. Формат:

```
┌─────────────────────────────────────────────────────────────────┐
│ Query #1: "japanese heart logo"                                  │
│ Routing: top-3 = [japanese (0.74), heart (0.51), kids (0.18)]   │
│                                                                   │
│  B          W           C2-eq      C2-soft    C2-norm           │
│  [img]      [img]       [img]      [img]      [img]              │
│  jp:NO      jp:YES      jp:YES     jp:YES     jp:YES             │
│  ht:NO      ht:NO       ht:YES     ht:YES     ht:YES             │
│  score:0    score:0.5   score:1.0  score:1.0  score:1.0          │
│                                                                   │
│  C3-soft    DB-C2-soft  Oracle                                   │
│  [img]      [img]       [img]                                    │
│  ...        ...         ...                                       │
└─────────────────────────────────────────────────────────────────┘
```

Каждый query = одна секция, 8 изображений в строку (или 2 строки по 4 если узкий экран). VLM scores под каждым изображением. **Routing info** в заголовке секции (top-3 LoRAs + gaps).

**Цвет фона**: лучшее изображение per query (по VLM score) — зелёная подсветка. Худшее — без подсветки.

### C. Aggregate metrics (CSV + summary в начале gallery)

```
condition  | mean_score | win_rate_vs_W | composition_success_rate
B          | 0.15       | 5%            | 5%
W          | 0.50       | -             | 50%  (только top-1 концепт)
C2-eq      | 0.78       | 75%           | 65%
C2-soft    | 0.82       | 80%           | 70%
C2-norm    | 0.81       | 78%           | 68%
C3-soft    | 0.74       | 70%           | 72%  (выше для 3-concept queries)
DB-C2-soft | 0.86       | 85%           | 78%  (де-биас помогает)
O          | 0.92       | 95%           | 90%  (ceiling)
```

**Definitions:**
- `mean_score`: средний VLM score по всем 20 queries
- `win_rate_vs_W`: % queries где condition >= W по VLM score
- `composition_success_rate`: % queries где **оба концепта присутствуют** (score = 1.0 для 2-concept, ≥0.67 для 3-concept)

---

## Success criteria

| Критерий | Threshold | Что значит |
|----------|-----------|------------|
| Best multi-LoRA condition mean_score > W | > 0.65 vs W=0.50 | Composition даёт прирост над winner-take-all |
| DB-C2-soft > наивные C2 варианты | +5pp в mean_score | De-bias валидирован, embedding bias есть |
| Best condition composition_success_rate ≥ 70% | — | Большинство compositional queries дают оба концепта |
| Best condition ≥ 80% от Oracle | mean_score ≥ 0.74 если O=0.92 | Близко к ceiling, дальнейшая оптимизация низкоприоритетна |
| Single-concept (Type C): W не хуже composition conditions | разница < 5pp | Composition не вредит на single-concept (важно для прода) |

---

## Risks / что может пойти не так

| Риск | Mitigation |
|------|------------|
| Multi-LoRA активация ломает FLUX (артефакты, дубли) | Сравнить визуально, попробовать lower weights (×0.7) |
| Embedding bias настолько силён что DB не помогает | Расширить atomic split heuristic, или прогнать через LLM для extraction |
| 3 LoRA одновременно = слишком много interference | C3-soft может проиграть C2 — это сам по себе результат |
| Oracle сложно подобрать (не очевидны «правильные» weights) | Использовать grid search [0.3-0.7] step 0.1 для O |
| FLUX seed жёстко доминирует над LoRA effect | Запустить 2-3 seeds per query, усреднить или взять best |

---

## Deliverables

1. **Code:** `~/Desktop/LLM/step0_inventory/run_composition_test.py` — main script
2. **Outputs:**
   - `~/Desktop/LLM/step0_inventory/_e2e_composition_results/gallery.html` ← **главный артефакт для review**
   - `_e2e_composition_results/scores.csv` — raw VLM scores
   - `_e2e_composition_results/aggregate.csv` — summary metrics per condition
   - `_e2e_composition_results/images/` — все сгенерированные изображения
3. **Report:** 1-2 страницы — что работает, что нет, рекомендация для production

---

## Resources

- GPU: RTX 3090 / 4090, ~6-8 часов inference (20 queries × 8 conditions × 1-2 seeds = 160-320 generations × ~10s each)
- VLM: Qwen2.5-VL-7B на Vadim CPU (slower, может быть лучше на GPU если есть)
- Cost: ~$30-40 GPU
- Time: 2-3 дня (1 день setup + 1 день run + 1 день analysis)

---

## Где это в общем плане

```
[Сейчас] Composition test ← ЭТА СПЕКА
              ↓
[После passing] разблокирует Era 6 composition
              ↓
[Параллельно W2] Auto-guild + Hierarchical routing experiment
              ↓
[После всё passing] Era 6 готова к интеграции в продакшн Jippy
```

**Что unlocks этот эксперимент:** возможность смешивать знания нескольких «специалистов» (= нескольких Jippy в federation) в один ответ. Это **killer feature** Era 6.
