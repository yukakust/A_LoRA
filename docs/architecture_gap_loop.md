# Architecture: Gap-Scraper-Retrain Loop

**Дата:** 2026-04-21
**Статус:** spec
**Связано:** `project_model_marketplace.md`, `project_contribution_economy.md`, `project_lora_learnings.md` (gap-scraper-retrain pattern)

---

## Идея

Система сама замечает где у неё дыры в знаниях (по low gap в роутинге) и автоматически закрывает их: триггерит сбор данных, тренинг нового LoRA, регистрацию в roster. **Замкнутая петля self-improvement без человека на каждой итерации.**

Открыто эмпирически в 19-LoRA Era 6 эксперименте (2026-04-21): gap = «громкость» голоса роутера, отражает редкость концепта vs приор базы.

---

## Components

```
[Gap Monitor]                    [Topic Bounty Board]
Логирует gap каждого              Публикует "нужны данные
запроса в продакшн                по теме X, reward Y"
       ↓                                  ↑
[Trigger Engine]                   [Crawlers / Contributors]
gap<0.2 на теме X >50              Юзеры или их боты
запросов за 7 дней                 собирают по bounty
       ↓                                  ↓
[Bounty Publisher] ──────────────→ [Data Pipeline]
Создаёт bounty с reward            Validation, dedup,
size = f(query volume)             quality check
       ↓                                  ↓
[Auto-Train Pipeline] ←────────── [Dataset Ready]
LoRA r=4-16, train, eval,          ≥100 quality items
register if pass eval                     ↓
       ↓                            [Royalty Stream]
[Registry Update]                  Each use of new LoRA
Новый LoRA в роутинге              → distribution per shares
```

---

## 1. Gap Monitor

**Что делает:** на каждый production query логирует:
- Query text (или semantic hash для приватности)
- Top-3 LoRAs и их gaps
- Чем закончилось: routed, abstained, fallback to baseline
- User feedback (если есть): 👍/👎

**Storage:** PostgreSQL table `gap_log` с rolling window 30 дней.

**Aggregation:** еженедельно cluster low-gap queries (DBSCAN на embeddings) → находим **темы** где gap систематически низкий.

---

## 2. Trigger Engine

**Условия для bounty:**
- Cluster size ≥ 50 queries за 7 дней
- Mean gap в кластере < 0.2 (не хватает специалиста)
- Cluster не покрывается существующим LoRA в roster
- Estimated demand × 4 недели > training cost (ROI positive)

**Выход:** structured bounty spec:
```yaml
topic: "skateboard / BMX / parkour logos"
example_queries: [...]  # 5-10 примеров
estimated_examples_needed: 200
reward_pool: 5000 GPU credits
deadline: 14 days
quality_criteria:
  - min_resolution: 512x512
  - format: PNG/JPG with transparent or solid bg
  - relevance_score (VLM): ≥0.7
```

---

## 3. Topic Bounty Board

UI на gpu.social где юзеры видят open bounties:
- Topic name + examples
- Reward pool + per-item rate
- Progress bar (collected / target)
- Time remaining
- Quality bar to pass

**Юзеры могут:**
- Submit data manually (upload)
- Connect Pinterest/Instagram crawler bots
- Subscribe на bounty notifications по своим интересам

---

## 4. Auto-Train Pipeline

При достижении target:
1. Validation: VLM eval каждого item, фильтр по threshold
2. Dedup: pHash (для images) или semantic hash (для text)
3. Train: LoRA r=4-16 (зависит от complexity), 500-2000 steps, RTX 4090 ~30-60 мин
4. Eval: hold-out test set (10% data) → loss + VLM coherence
5. Capability vector extraction (Arrow)
6. Если pass eval → register в Model Registry

**Если fail:** bounty продлевается, нужно больше данных или улучшение качества.

---

## 5. Reward Distribution (расширение 70/20/10)

Когда новый LoRA используется в проде, royalty stream от каждого вызова делится:

| Роль | Доля | Логика |
|------|------|--------|
| **Gap identifiers** | 5% | Те, чьи запросы первыми засчитаны в trigger cluster (top-N контрибьюторов в trigger) |
| **Data contributors** | 50% | По объёму × quality score каждого contributor |
| **LoRA trainer** | 30% | Тот кто запустил тренинг (или система если автоматически) |
| **Validators** | 5% | Те кто прошёл eval/QA пайплайн |
| **Platform (gpu.social)** | 10% | Infra cost + bounty seed pool |

**Replication contribution_economy patterns:**
- Lock price-up once registered (как в forks)
- Royalty auto-scales down если ancestor LoRA меняется
- Multi-fork attribution: если LoRA fork'ается, ancestors получают 20/10 split

---

## 6. Lifecycle States

```
[OPEN] → bounty published, accepting contributions
   ↓
[FUNDING] → reward pool grows from interested users (optional)
   ↓
[COLLECTING] → contributors submit data
   ↓
[VALIDATING] → quality checks
   ↓
[TRAINING] → auto-train pipeline running
   ↓
[EVAL] → testing the new LoRA
   ↓
[REGISTERED] → in production roster, royalty flowing
   ↓
[DEPRECATED] → если другой LoRA лучше или topic outdated
```

---

## 7. Privacy considerations

- Production query logs **никогда не содержат raw user data** — только embeddings hashes + gaps
- Bounty topics — публичные (это feature: анонимный коллективный intelligence)
- Contributors данные принадлежат им → upload требует явного opt-in license
- LoRA веса публичны (это marketplace), но training data — хранится отдельно с access control

---

## 8. Anti-abuse

| Атака | Защита |
|-------|--------|
| Спам queries для триггеринга bounty на свою тему | Cluster требует diverse user_ids (≥10 unique users) |
| Submit junk data для reward | Quality threshold + reputation system + slashing для repeat offenders |
| Train low-quality LoRA для grab royalty | Auto-eval gate + human review для top earners |
| Sybil attacks (фейковые юзеры) | Proof-of-work для critical actions + GPU compute history check |

---

## 9. Integration points

**С Jippy production:**
- Gap Monitor подключается к каждому inference call
- Bounty notifications появляются в Jippy UI ("ваш Jippy не знает X, помогите собрать данные?")
- Royalty earning виден в user dashboard

**С Marketplace (gpu.social):**
- Bounty Board как новая section
- Auto-registered LoRAs отмечены badge "auto-grown"
- Filter по lineage (manual vs auto-grown)

**С Contribution Economy:**
- Reward distribution использует тот же royalty engine
- Lineage tracking (кто triggered → кто contributed → кто trained)
- Lock-price rules применяются

---

## 10. MVP scope (первая реализация)

| Компонент | MVP | V1 | V2 |
|-----------|-----|-----|-----|
| Gap Monitor | ✅ logging + manual cluster review | Auto cluster | + анализ feedback |
| Trigger Engine | Manual approval bounties | Threshold-based auto | + ROI prediction |
| Bounty Board | Простой list + manual upload | Full UI + crawler integration | + funding mechanic |
| Auto-Train | Manual pipeline | Auto на triggered bounty | + multi-LoRA composition test |
| Reward dist | Manual distribution | Smart contract / automated | + slashing |

**MVP timeline:** 2-3 недели после passing composition test.

---

## 11. Связь с другими подсистемами

- **Era 6 (federation):** gap-loop работает на агрегированных production gaps. В federation версии — каждый Jippy свой gap log + global aggregation для cross-user trends
- **Era 7 (guilds):** новые auto-grown LoRAs автоматически кластеризуются в guilds (auto-guild experiment)
- **Sonar pilot:** для personal Jippy gap-loop работает локально (твой Jippy не знает о теме X → suggest collect data)
- **Training Rules:** auto-train pipeline должен соблюдать `docs/TRAINING_RULES.md` (base whitelist, eval, metadata)

---

## 12. Open questions

1. **Cluster threshold** — 50 queries / 7 days хорошее значение? Скорее всего нужна tuning по реальным production logs
2. **Bounty pricing** — как фиксировать reward pool? Auction style (max bid) vs fixed schedule?
3. **Quality eval** — VLM eval достаточно для visual? Для text — нужен LLM judge?
4. **Conflict resolution** — что если 2 trainers одновременно работают над одним bounty? First-to-pass-eval wins?
5. **Topic granularity** — насколько узкие темы триггерить? «skateboard logo» vs «extreme sport logos»? Trade-off precision vs roster size
