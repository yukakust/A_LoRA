# A_LoRA — Architecture & Strategy Snapshot

> **Frozen public snapshot** of [yukakust/pt-moe-research](https://github.com/yukakust/pt-moe-research) (private), captured 2026-04-29.
> Live development continues in the private repo. This mirror **will not be updated** — it's a point-in-time showcase of architecture and strategy.

---

## What's inside

This is the architecture, strategy, and implementation thinking behind **Jippy** — a personal AI that builds a real model of *you* (your voice, your knowledge, your relationships) and runs locally on your device.

The thesis: don't compete with frontier models on intelligence — compete on **architectural surface area** they structurally can't reach (personalization, edge inference, sensor grounding, user ownership, continuous learning).

### Start here

- **[`INDEX.md`](INDEX.md)** — navigation map of the whole repo
- **[`docs/jippy_architecture_final.md`](docs/jippy_architecture_final.md)** — full product spec (~1700 lines)
  - §17 Memory Architecture — 5-layer memory model mapped onto Tulving/Squire taxonomy (episodic / semantic / procedural / consolidation)
  - §17.5.1 Tiered User Profile — fact / inference / hypothesis with active testing & decay
  - §16 Proximity Mesh & LoRA Hunt — physical-presence federated learning between phones
- **[`FRONTIER_ATTACK.md`](FRONTIER_ATTACK.md)** — strategic doc: 8 architectural holes in OpenAI / Anthropic / Gemini we're attacking, plus what we honestly *can't* attack
- **[`docs/IMPLEMENTATION_PLAN.md`](docs/IMPLEMENTATION_PLAN.md)** — phase-by-phase backend roadmap, with done/not-done markers

### Active narrow specs (in `docs/`)

Topic-specific feature specs — TRAINING_RULES, experiment_arrow_validation, experiment_composition_test, feature_spec_hybrid_search, feature_spec_import_ai_history, feature_spec_sonar, pilot_sonar_founder, architecture_gap_loop.

### Archive

Historical snapshots of earlier thinking — kept for context, no longer the source of truth: `JIPPY_TZ.md`, `JIPPY_AI_ARCHITECTURE.md`, `GPU_EVOLUTION.md`, `SOLUTIONS.md`, `PROBLEMS.md`, `COMPETITORS.md`, `UNDERDOGS.md`, `EXPERIMENTS.md`, `PRODUCT_FLOW.md`, `apps.md`, `ataturk_turkish_data_sources.md`, `georgian_cuisine_data_sources.md`.

---

## Why this exists

Open-sharing the architecture serves two goals:
1. **Accountability** — the design is on record. If we ship something different, you can hold us to it.
2. **Conversation** — sharper minds than mine have already thought about parts of this. We're better off being readable.

What's NOT here:
- Working code (lives in `yukakust/jippy`, private)
- Personal data — no TG history, no profiles, no encrypted DBs
- Operational secrets — no IPs, no credentials, no infrastructure details

---

## Status

**Build**: backend MVP shipped (Phase 0–4 done as of 2026-04-29). Tiered Profile, Knowledge Map with NER, multi-signal closeness scoring, parallel-fetch harness — all implemented and wired into a working web UI. See `docs/IMPLEMENTATION_PLAN.md` "Done so far" for current state.

**Next**: device port (iOS/Android), sensor pipeline, LoRA Hunt v0, Mindful Cart (food→outcome correlation feature).

---

## Want to dig deeper?

This is a **frozen snapshot** — the active work continues in the private repo, with new sprint docs landing every 1-2 weeks (Self-Portrait pipeline, Insight Detectors, Story Timeline, Hierarchy & Ownership graphs, …).

If anything here resonates and you want to **chat / contribute / get private-repo access**:
- 💬 Open a GitHub Discussion on this repo — public, threaded, async-friendly
- 📩 Email **kustyuka@gmail.com**
- ✈️ Telegram **@yuka_k**

I'm especially interested in talking to people working on:
- Edge inference (CoreML / ONNX RT Mobile / on-device LoRA)
- Personal-data sovereignty / federated approaches
- Memory architecture beyond simple RAG (Tulving/Squire taxonomy, episodic vs semantic)
- Anything in §17 of `docs/jippy_architecture_final.md`

---

## License

Documents only. No code in this repo. Specs and strategy are shared under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — free to read, share, adapt, with attribution.

---

*Maintained by [@yukakust](https://github.com/yukakust). Private working repo: `yukakust/pt-moe-research`. Backend: `yukakust/jippy`.*
