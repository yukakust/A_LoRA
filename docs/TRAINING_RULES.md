# TRAINING_RULES — Community Model Training Rules for gpu.social

**Цель:** правила тренинга моделей чтобы они автоматически попадали в наш роутинг-пул и могли смешиваться с серверными backbone моделями для лучших ответов пользователям.

Если модель не соответствует этим правилам — она работает только standalone, без интеграции в Jippy/backbone composition.

---

## 0. TL;DR (минимум который нужен)

1. Базовая модель — из **разрешённого списка** (см. §2)
2. Размер LoRA адаптера ≤ 200MB (или full model ≤ 4GB INT4)
3. Заполнен `model_card.yaml` (см. §4)
4. Output формат — **structured JSON или Markdown секции** (см. §5)
5. Прогнан стандартный eval, метрики приложены (см. §7)
6. Capability vector извлечён или указаны теги (см. §9)

---

## 1. Зачем эти правила

Когда пользователь задаёт вопрос Jippy, наша система:
1. Embeds запрос → ищет в Model Registry подходящих specialists
2. Загружает 1+ specialist + backbone в память
3. Specialist даёт hint → backbone генерирует ответ

**Чтобы это работало:**
- Specialist должен быть совместим с backbone (общая база или совместимый токенизатор)
- Output specialist должен быть **парсабельным** (backbone должен понять что injected)
- Метаданные должны позволить роутеру понять **когда вызывать** эту модель

---

## 2. Разрешённые базовые модели

### Text-text (для обычного вопрос-ответ)
| Base | Param | Где используется |
|------|-------|------------------|
| Qwen3.5-0.8B | 0.8B | Phone-side specialists |
| Qwen3.5-4B | 4B | Server-side specialists, MVP backbone |
| Qwen3.5-14B | 14B | Heavy specialists |
| Llama-3.5-8B | 8B | English-focus specialists |
| Mistral-Small-3 | 24B | EU-focus specialists |

### Multimodal (text + image input)
| Base | Param | Modality |
|------|-------|----------|
| Qwen3.5-VL-0.8B | 0.8B | image→text on phone |
| Qwen3.5-VL-4B | 4B | image→text server |
| InternVL-3 | 8B | image+video understanding |

### Image generation (text → image)
| Base | Где используется |
|------|------------------|
| FLUX.2 Schnell | Quick image gen specialists |
| SDXL Lightning | Style-specific specialists |

**Если нужна другая база — создай PR в этот файл с обоснованием.**

---

## 3. Типы модальностей

Каждая модель должна задекларировать одну из:

```yaml
modality:
  input: ["text"]                    # text-text
  output: ["text"]
  
  input: ["text", "image"]           # multimodal input
  output: ["text"]
  
  input: ["text"]                    # image gen
  output: ["image"]
  
  input: ["audio"]                   # speech
  output: ["text"]
```

Роутер будет выбирать specialists только подходящей модальности под запрос пользователя.

---

## 4. Required Metadata: `model_card.yaml`

Каждая опубликованная модель ОБЯЗАНА иметь `model_card.yaml` в корне:

```yaml
# === Identity ===
name: "med-qa-russian-v2"
version: "1.0.0"
author: "@username"
license: "Apache-2.0"  # или MIT, CC-BY-SA, custom
created_at: "2026-04-17"

# === Base ===
base_model: "Qwen3.5-4B"
base_model_revision: "main"
adapter_type: "lora"         # lora | full | dora | ia3
adapter_size_mb: 87

# === Modality ===
modality:
  input: ["text"]
  output: ["text"]

# === Capability ===
domain: ["medical", "russian", "qa"]      # тематика
languages: ["ru", "en"]
task_types: ["question-answering", "diagnosis-explanation"]

# === Training ===
training:
  dataset_size: 12500              # количество примеров
  dataset_hash: "sha256:abc123..."  # для дедупликации
  epochs: 3
  steps: 4500
  data_sources:
    - "MedQA-RU translated"
    - "RuMedBench"

# === Eval ===
eval:
  benchmark: "med_benchmark_ru_v1"   # стандартный или custom
  score: 0.74                        # win-rate vs baseline
  baseline: "Qwen3.5-4B vanilla"
  eval_examples: 500

# === Output Format ===
output_format: "structured_hint"     # см. §5
hint_token_budget: 256

# === Capability Vector (auto или manual) ===
capability_vector:
  source: "arrow_extracted"          # или "manual_tags" или "embedded_descriptions"
  vector_dim: 768                    # размер embedding
  vector_file: "capability.npy"

# === Cost ===
cost:
  vram_mb: 850
  inference_latency_ms_p50: 180
  warm_load_time_ms: 2400
```

---

## 5. Output Format Requirements

Specialist должен выдавать output в одном из утверждённых форматов чтобы backbone мог его inject в свой prompt:

### Format A: Structured Hint (рекомендуется для text)

```json
{
  "confidence": 0.85,
  "facts": [
    "Метформин — препарат первой линии при СД2",
    "Доза 500-2000 мг/сутки"
  ],
  "context": "Пациент 55 лет с впервые выявленным СД2",
  "sources": ["clinical_guidelines_ru_2025"],
  "warnings": ["Не назначать при СКФ < 30"]
}
```

### Format B: Markdown Sections

```markdown
## Confidence
0.85

## Key Facts
- Fact 1
- Fact 2

## Reasoning
Brief reasoning here

## Caveats
What to be careful about
```

### Format C: Image Description (для multimodal input → text output)

```json
{
  "confidence": 0.9,
  "description": "Рентген грудной клетки, видна инфильтрация в правой нижней доле",
  "objects_detected": ["lung", "ribs", "infiltrate"],
  "abnormalities": [{"type": "infiltrate", "location": "right_lower_lobe", "confidence": 0.78}]
}
```

### Format D: Image Generation Output

Стандартный image binary + metadata JSON с `seed`, `steps`, `cfg`, `prompt_used`.

**Если модель выдаёт свободный текст без структуры — она НЕ может быть injected в backbone, только standalone.**

---

## 6. Training Data Rules

### Обязательно
- **Dataset hash** в `model_card.yaml` — для дедупликации между моделями
- **Минимум 1000 примеров** (меньше = шум, не примем в Registry)
- **Eval split** — минимум 10% данных отложить в holdout, не использовать в train

### Запрещено
- Тренинг на данных пользователей **без их согласия** (privacy violation)
- Тренинг на копирайтном контенте без лицензии
- Тренинг на данных где встречается личная информация (PII) без анонимизации

### Рекомендовано
- **Format consistency**: все training examples в одном output формате (см. §5). Иначе модель будет выдавать вперемешку.
- **Negative examples**: примеры "я не знаю" для запросов вне домена (так модель не будет галлюцинировать когда роутер ошибётся)
- **Calibration**: confidence в output должен коррелировать с правильностью

---

## 7. Eval Requirements

### Минимум для попадания в Registry
- Прогон на **стандартном бенчмарке** для своего домена (см. список ниже)
- Win-rate ≥ baseline (vanilla base model) на ≥ 60% задач
- Eval results файл `eval_results.json` приложен

### Стандартные бенчмарки (выбрать релевантный)
| Domain | Benchmark | Где взять |
|--------|-----------|-----------|
| Medical | MedQA, MedMCQA, RuMedBench | HuggingFace |
| Math | GSM8K, MATH-5, AIME | HuggingFace |
| Code | HumanEval, MBPP, BigCodeBench | EvalPlus |
| Translation | FLORES-200, NLLB-eval | HuggingFace |
| General QA | MMLU, ARC, TruthfulQA | LM Eval Harness |
| Multimodal | MMMU, MMBench, ChartQA | HuggingFace |

**Custom бенчмарки**: можно добавить свой, но должен быть public + воспроизводимый.

### Формат `eval_results.json`
```json
{
  "benchmark": "MedQA",
  "your_model": {
    "accuracy": 0.74,
    "n_examples": 500,
    "elapsed_sec": 312
  },
  "baseline": {
    "model": "Qwen3.5-4B",
    "accuracy": 0.61
  },
  "win_rate": 0.74,
  "categories": {
    "diagnosis": 0.78,
    "treatment": 0.71,
    "pharmacology": 0.69
  }
}
```

---

## 8. Size & Performance Limits

| Тип | Max size | Reason |
|-----|----------|--------|
| LoRA adapter | 200MB | Быстрая загрузка в память |
| Full fine-tune | 4GB INT4 | Помещается в общий VRAM pool |
| Quantization | INT4 / INT8 | INT4 предпочтительно для скорости |
| Inference latency p50 | < 500ms | Иначе слишком медленно для composition |
| Warm load time | < 5 sec | Иначе пользователь будет ждать |

**Если модель больше — она пойдёт в "heavy specialists" tier, доступна только премиум пользователям с дополнительной оплатой.**

---

## 9. Capability Vector (для роутинга)

Роутер выбирает модель по сходству embedding запроса с capability vector модели.

### Способ 1 (рекомендуется): Arrow auto-extraction
- Мы автоматически прогоняем твою модель через **Arrow extraction pipeline**
- Получаем capability vector из LoRA весов
- Точность роутинга: 97% от oracle (валидировано в Phase 0/1)
- **Ничего делать не надо**, кроме publish

### Способ 2: Manual tags + descriptions
Если Arrow extraction не работает (например, full fine-tune):

```yaml
capability_vector:
  source: "manual_tags"
  description: "Russian medical question answering, focus on clinical practice and pharmacology. Trained on RuMedBench and translated MedQA."
  example_queries:
    - "Какая первая линия терапии при гипертонии у пожилых?"
    - "Противопоказания к метформину"
    - "Дифдиагностика боли в груди"
```

Мы embedding'нем description + example_queries и используем как capability vector.

### Способ 3: Embedded descriptions (для multimodal)
Для image моделей:
```yaml
capability_vector:
  source: "embedded_descriptions"
  example_images: ["xray.jpg", "mri.jpg", "ct.jpg"]
  example_descriptions: [...]
```

---

## 10. Publishing Checklist

Перед публикацией модели проверь:

- [ ] `model_card.yaml` заполнен и валиден
- [ ] `eval_results.json` приложен
- [ ] Output формат соответствует §5
- [ ] Размер модели в лимитах §8
- [ ] License указана и совместима
- [ ] Dataset hash указан, без PII
- [ ] Capability vector файл (или manual tags) приложен
- [ ] Прогнан smoke test (50 запросов из примеров) — без ошибок

После прохождения checklist твоя модель автоматически попадает в **Model Registry** и становится доступна для роутинга в Jippy/backbone composition.

---

## 11. Что получает автор

- **Use-based вознаграждение**: каждый вызов твоей модели в composition приносит долю (см. Contribution Economy)
- **Quality bonus**: модели с высоким win-rate получают приоритет в роутинге = больше использований
- **Discoverability**: твоя модель появляется в поиске gpu.social с тегами и benchmark scores

---

## 12. Что мы оставляем за собой

- Право убрать модель из Registry если она:
  - Получает массовые negative feedback от пользователей
  - Замечена в галлюцинациях / небезопасных ответах
  - Нарушает копирайт / privacy
- Право обновлять формат `model_card.yaml` (с миграцией)
- Право добавлять новые eval benchmarks как обязательные

---

## 13. FAQ

**Q: Можно ли тренировать на нашей платформе и сразу публиковать?**
A: Да, gpu.social предоставит "Publish to Registry" кнопку прямо из training UI.

**Q: Что если моя модель не проходит eval threshold?**
A: Будет доступна как "experimental" — не в основном роутинге, но пользователи могут её subscribe вручную.

**Q: Можно ли private модель только для своих пользователей?**
A: Да, при публикации указать `visibility: "private"` + список авторизованных user_id.

**Q: Как часто обновлять модель?**
A: Версионирование semver. Major changes (несовместимый output формат) — bump major. Patch для improvements.

**Q: Что если я хочу тренировать на другой базовой модели не из списка §2?**
A: Создай PR с обоснованием — мы рассмотрим. Критерии: licensing OK, fits VRAM tier, есть spec на токенизатор/output.
