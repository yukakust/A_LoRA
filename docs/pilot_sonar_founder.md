# Pilot: Sonar Founder Dogfooding

**Дата создания:** 2026-04-21
**Статус:** ready-to-execute (стартует после passing Arrow validation test)
**User #1:** Yuri Kustov (<founder email>)
**Связано с:** `feature_spec_sonar.md`, `experiment_arrow_validation.md`, `jippy_architecture_final.md`

---

## Цель

Запустить минимальную (scrappy) версию Sonar на собственных данных user #1 чтобы:
1. Валидировать что Jippy с full personal context реально полезен (не gimmick)
2. Найти edge cases в parsers/connectors на real data
3. Получить feedback по UX и privacy intuitions
4. Подготовить decision: productize Sonar или pivot

---

## Scope: Scrappy, не Production

Production Sonar = 3 мес. Pilot = **3-4 недели** за счёт упрощений:

| Compromise | Production | Pilot |
|------------|------------|-------|
| UI | React app, dashboards | Jupyter notebook + CLI |
| Auth | OAuth flows для всего | Manual exports + 1-2 OAuth (Gmail) |
| Connectors | Plugin framework | Хардкод под user'а |
| Privacy tiers | UI toggles per source | Всё на Vadim server (приватный) |
| Live sync | Webhooks + cron | One-shot scan |
| Knowledge Map viz | Полноценная карта | Простая matplotlib визуализация |

---

## Источники для pilot

| Источник | Как получить | Объём (~) | Приоритет |
|----------|--------------|-----------|-----------|
| Mac Documents/Desktop | rsync на Vadim | 1-50 GB | High |
| Code repos (Jippy, pt-moe-research) | git clone | 5-20 GB | High |
| Gmail (<founder>@) | OAuth + IMAP fetch | 1-10 GB | High |
| Telegram | Built-in JSON export | 0.5-5 GB | High |
| Claude/ChatGPT histories | ZIP exports (`feature_spec_import_ai_history.md`) | 0.1-1 GB | Medium |
| Apple Notes | macOS export | 0.1 GB | Medium |
| Browser history | SQLite копия из Chrome/Safari | 0.1 GB | Medium |
| Photos | iCloud/Photos export + VLM caption | 10-100 GB | Low (тяжёлый) |
| Jippy own dev logs | git + Claude memory + transcripts | small | Meta-bonus |

**Минимум для MVP pilot:** первые 4 источника. ~10-50 GB суммарно.

---

## Timeline (3-4 недели)

### Week 1 — Data Collection
- Setup: dedicated folder на Vadim server `/home/jippy/sonar_pilot/`
- Mac Documents → rsync (с .gitignore-style фильтром: skip cache/build/node_modules)
- Code repos → git clone (или rsync рабочих копий)
- Gmail → OAuth setup, fetch via IMAP в .mbox формате
- Telegram → export JSON через Desktop client
- Claude/ChatGPT → manual ZIP downloads
- **Deliverable:** raw data на Vadim, ~10-50 GB

### Week 2 — Ingestion Pipeline
- Parser per source type:
  - Files: PDF (PyMuPDF), DOCX (python-docx), MD/TXT (raw), code (raw + AST)
  - Email: MIME parse, HTML→text, attachment extraction
  - Telegram JSON: messages → Q&A pairs (questions = твои сообщения, answers = ответы или контекст)
  - Claude/ChatGPT JSONs: уже спроектировано в спеке
- Normalizer → unified Q&A schema (`jippy_architecture_final.md`)
- Embeddings: `multilingual-e5-small` (118MB)
- Index: PostgreSQL + pgvector + FTS (`feature_spec_hybrid_search.md`)
- **Deliverable:** indexed 100K-500K Q&A pairs, searchable

### Week 3 — Knowledge Map + Jippy Chat
- Auto-cluster embeddings (HDBSCAN или K-Means → labeling через top terms)
- Простая visualization: matplotlib scatter с кластерами + labels
- Chat interface: Telegram bot или CLI script
- Backbone: Kimi K2.5 API (или Qwen3.5-4B локально на Vadim позже)
- Hint injection из top-K hybrid search results на каждый запрос
- **Deliverable:** working Jippy с full personal context, можно общаться

### Week 4 — Iteration & Report
- User testing самим собой каждый день
- Лог: что работает / что фейлится / что неожиданно полезно
- Bug fixes по горячим находкам
- Bonus test: try Era 6 self-routing — если есть кластеры по темам в Knowledge Map, протестировать **routing между ними** на compositional queries (например «как мне совместить Jippy архитектуру с federated knowledge?» — должно потянуть 2 кластера)
- **Deliverable:** report с findings + decision recommendation

---

## Метрики успеха pilot

| Метрика | Target | Что значит |
|---------|--------|------------|
| Reaction time | < 5s end-to-end | UX viable |
| "Useful response" rate (self-rated) | ≥ 60% запросов | Полезно, не gimmick |
| Coverage: how often "I don't have data on this" | < 30% | Источники покрывают типичные queries |
| Privacy comfort | Self-assessment 1-5 | Реально ли чувствуется ОК |
| Edge cases found | logged | Знаем что чинить для production |
| Era 6 self-routing accuracy | top-2 recall ≥ 70% на compositional self-queries | Bonus validation |

---

## Test queries для pilot (примеры)

Запросы в стиле «то что Google/Claude не могут ответить»:

- «Что я обсуждал с Vadim'ом про FLUX тренинг 2 недели назад?»
- «Какие LoRA эксперименты я делал в Phase 0?»
- «Что я писал в Telegram про Jippy positioning?»
- «Покажи мой код где я делал hybrid search»
- «Какие были проблемы с RTX 3090 в марте?»
- «Compose: соединить мою архитектуру Jippy с Sonar выводами»
- «Мои заметки про Era 6 federation — суммаризируй»

Если Jippy отвечает на это лучше чем Claude/ChatGPT (потому что у них нет доступа к твоим данным) — pilot succeed.

---

## Риски и mitigation

| Риск | Mitigation |
|------|------------|
| Scope creep («ещё этот источник, и этот...») | **Жёсткий time-box: 4 недели hard stop** |
| Demo trap (строим для красивого видео, не для продукта) | Ты как реальный user, не «играешь юзера» |
| Personal data leak | Всё на Vadim server (private), backup encrypted, no third-party calls кроме Kimi API (и то только embeddings + hints, не raw) |
| Founder bias (работает только на тебе) | После pilot — 3-5 friends pilot для проверки generalization |
| Data volume overwhelms Vadim | Пресказание: 50GB должно влезть. Если больше — sample/filter |
| Embedding cost для миллионов Q&A | `multilingual-e5-small` локально, бесплатно |

---

## Ресурсы

| Ресурс | Что нужно | Доступно |
|--------|-----------|----------|
| Server | Vadim Ubuntu | ✅ есть |
| Storage | ~100 GB свободного | проверить |
| GPU | Не критична для pilot (без VLM Photos) | RTX 3090 на Vast если понадобится |
| Embedding model | `multilingual-e5-small` | download free |
| Backbone API | Kimi K2.5 | ~$10-30 для testing |
| Time | 1 dev × 3-4 недели | TBD |
| Total cost | ~$10-30 | + время |

---

## Sequence — где это в общем плане

```
[Сейчас]   Arrow validation test on logos (1-2 нед, в работе)
              ↓
[После passing] Sonar pilot на user #1 (3-4 нед)
              ↓
[После passing pilot] 3-5 friends pilot (4-6 нед)
              ↓
[После passing] Productize Sonar (3 мес, MVP)
              ↓
[Параллельно] Era 6 federated extension validation
```

**Каждый шаг = decision gate.** Если Arrow проваливается — Sonar pilot всё равно полезен (без compositional routing, но с personal context). Если pilot не вызывает «вау» — переоцениваем product direction до 3 мес инвестиции в production.

---

## Что должно быть готово до старта pilot

- [x] Sonar specification (`feature_spec_sonar.md`) — есть
- [x] Hybrid search spec (`feature_spec_hybrid_search.md`) — есть
- [x] Q&A schema (`jippy_architecture_final.md`) — есть
- [ ] Arrow validation results — ждём
- [ ] Vadim server disk space check
- [ ] Назначить dev на pilot
- [ ] Backup plan для personal data перед stress testing

---

## Decision criteria после pilot

| Outcome | Action |
|---------|--------|
| ≥ 60% useful responses + WOW моменты | **Go to friends pilot** → потом productize |
| 40-60% useful + edge cases решаемы | Iterate pilot ещё 2 нед, потом friends |
| < 40% useful или фундаментальные блокеры | **Stop, переосмыслить product direction** |
| Edge case: работает но не «стоит того» | Pivot фокус (например только code/work assistant) |
