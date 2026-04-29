# Jippy AI Architecture — Final Design (April 2026)

> Status: **Phase 0 COMPLETE on vast.ai (2026-04-11). Phase 1 (multi-LoRA composition) running**.
> Last updated: 2026-04-12
> Companion document: `JIPPY_TZ.md` (product spec), `EXPERIMENTS.md` (PT-MoE V5 fusion history)
>
> ## Mental model (corrected 2026-04-11)
> **Jippy is ALWAYS a phone-side LoRA** — it travels with the user. It pairs with
> a backbone that scales by user "level":
> - **Offline / Level 1-2**: Jippy LoRA + SmolLM 1.7B (phone)
> - **Online / Level 3+**: Jippy LoRA + top OSS model (server)
> - **Endgame**: cascade to OpenAI / Anthropic API only on queries where the user
>   has no personal context advantage
>
> The "level up" is the SERVER backbone getting bigger. The LoRA stays the same
> (or grows with more user data). Two synchronized LoRAs are needed (one per
> backbone family) since LoRA weights can't span hidden-state spaces.
>
> ## Phase 0 result: ✅ Arrow Routing validated, but loss-eval misled us
>
> ### Routing discrimination (smoke real, ckpt-1600, N=30 per set)
> | Set | Mean cosine | Std |
> |---|---|---|
> | Python | 0.217 | 0.028 |
> | MMLU   | 0.138 | 0.011 |
> | GSM8K  | 0.141 | 0.012 |
>
> Margin Python − OOD: **+0.077 to +0.080**. Calibrated gate:
> `sigmoid((cos − 0.158) / 0.02)`. **Balanced accuracy 95%** on N=30 per set.
>
> ### Variant comparison — A-E loss results (final adapter)
> ```
>                        4-bit                    FP16
> Variant               python  mmlu  gsm8k       python  mmlu  gsm8k
> A Baseline            1.86    1.58  1.74        1.92    2.14  1.82
> B Always-on LoRA      0.48    0.39  0.71        0.48    0.45  0.70
> C Arrow routing       0.48    0.40  0.70        0.48    0.46  0.69
> D Oracle gate         0.48    1.58  1.74        0.48    2.14  1.82
> E Fixed alpha=0.5     0.52    0.55  0.71        0.53    0.67  0.81
> ```
>
> **Surprise:** by loss, Always-on LoRA wins everywhere — even OOD. Oracle gate
> (LoRA off on OOD) is 3-4× WORSE on mmlu/gsm8k. This contradicted our V5
> hypothesis that "LoRA hurts OOD".
>
> ### Hole #1 closed: 4-bit quantization is NOT the cause
> FP16 baseline is no better than 4-bit baseline (actually slightly worse on
> MMLU: 2.14 vs 1.58). Quantization is not driving the gap.
>
> ### Hole #3 closed: Loss is misleading on OOD evaluation
> Generation-quality eval (jaccard1 on first line vs gold, MAX_NEW=1024, N=10):
>
> | Domain | Loss B/A | Jaccard B/A | Reality |
> |---|---|---|---|
> | python | ×3.9 | **×30+** (0.02 → 0.71) | ✅ Real strong boost |
> | mmlu | ×4.0 | ×1.2 | ⚠️ FORMAT MIMICRY only |
> | gsm8k | ×2.4 | ×6 | ⚠️ Format buff, not knowledge |
>
> **Loss rewards format-matching** ("Sure! Here is..." matching gold's opener)
> without rewarding correct content. On structured short-answer datasets the
> loss gap is dominated by format, not knowledge. The original V5 hypothesis
> "LoRA hurts OOD → need routing" is technically false but practically still
> useful: routing avoids unnecessary alpha=1 on OOD (small benefit) and is
> essentially free to compute.
>
> ### Open holes
> - **Hole #2**: train raw-code LoRA without instruction format → confirm format-mimicry effect disappears
> - **Hole #3b**: regenerate gsm8k with MAX_NEW=1024 → see if LoRA actually solves math when not cut off mid-solve
> - **Hole #4** (Phase 1): train Math LoRA + multi-LoRA composition test (running on vast.ai)

---

## TL;DR

Personal LoRA(s) on the phone, paired with a backbone that scales by user level. The personal stack is glued together by a **6-layer pipeline** of published, training-free or near-free components from 2024-2026 research:

0. **Format LoRA always-on** *(NEW post-Phase-0)* — small adapter trained on diverse instruction-following data; gives format buff to OOD without specializing
1. **RAG retrieval** — pull facts from user's knowledge_bank into the prompt
2. **Arrow Router** — SVD prototype of each specialist LoRA decides which to fire; computes per-LoRA alpha
3. **FedALT Mixer** — composes top-K specialist LoRAs with Arrow weights into the backbone
4. **Self-REF Confidence Token** — personal model emits a `[CONF]` logit signaling "I am sure / I don't know"
5. **Cascade Escalation** — if Self-REF says "not sure", drop the personal output and use API (Anthropic/OpenAI) instead

Every component is documented in published papers (citations below). The user's vision of "narrow personal specialist + general server backbone" is the **dominant 2024-2026 paradigm**, validated by Apple Intelligence in production.

---

## Why This Architecture (vs PT-MoE V5 Fusion Failure)

PT-MoE V5 fusion failed because it used a **learned `nn.Linear` gate** trained on token-level loss across uniformly-sized tracks. The MoErging Survey (Yadav et al., arxiv 2408.07057) explicitly identifies this as the textbook failure mode:

> "When adapters have clear domain specialization, retrieval routing beats learned gates because learned gates need differential loss signal which single-domain adapters don't provide."

Our V5 had:
- Track 0 (base) winning 93.2% of examples
- Donor tracks (Gemma/Phi/Qwen) winning only 6.8%
- proj_in/proj_out projections never learning to extract donor knowledge
- Gate collapsing to "always pick base"

The 5-layer architecture below avoids learned gates entirely (Layers 2 and 4 use training-free signal extraction; Layer 5 is hard switching).

---

## The 6-Layer Stack

### Layer 0 — Format LoRA (always-on) *(NEW post-Phase-0)*

**Purpose:** Give every output a consistent, helpful, instruction-following style — without specializing in any particular domain.

**Why it exists:** Phase 0 generation eval revealed that the Python LoRA's "OOD win" was format mimicry: it learned "respond concisely in code-like form", which loss-rewards on any structured-output dataset (MMLU's "(A)/(B)/(C)" format, GSM8K's `#### N` final-answer format). On hard MMLU questions both baseline and LoRA fail equally — there is no real knowledge transfer. So instead of pretending the specialist LoRA helps OOD, we admit it doesn't and add a **dedicated** small format LoRA that's always on.

**Implementation:**
- Train one small LoRA (r=8 or r=16) on a diverse mix of instruction datasets (Alpaca, Dolly, OASST, etc.) — explicitly NOT specialized
- Goal: learn to follow chat format, terminate cleanly, follow length cues, respect "answer only" / "explain step-by-step" instructions
- Always loaded, scaling fixed at 1.0
- Size target: ~30-50 MB (similar to specialist LoRAs)

**Cost:** Negligible — same as any LoRA forward pass; no routing decision needed.
**Open question:** Does Format LoRA + specialist LoRA composed via FedALT actually beat each alone? Phase 1 will test this with two specialists, then Phase 2 will add Format LoRA to the mix.

---

### Layer 1 — RAG Retrieval

**Purpose:** Look up relevant facts from user's knowledge_bank before answering.

**Implementation:**
- Embedding model: `sentence-transformers/all-MiniLM-L6-v2` (25 MB, CPU)
- Storage: Postgres + pgvector
- Top-K = 3 most similar facts retrieved by cosine similarity
- Retrieved facts injected into prompt as context

**Cost:** ~5ms per query
**Boost:** +14.92% over baseline (LaMP benchmark, ACL 2024)
**Source:** [LaMP: Large Language Models personalization benchmark](https://lamp-benchmark.github.io/)

---

### Layer 2 — Arrow Routing

**Purpose:** Decide BEFORE generation whether the personal LoRA is relevant to this query.

**Implementation (validated config from Phase 0 smoke v2):**
- At LoRA load time, for every wrapped Linear:
  - Compute `W_delta = B @ A` (B: d_out×r, A: r×d_in)
  - `v_prototype = top right singular vector of W_delta` via `torch.svd_lowrank(W_delta, q=1)`
  - Store `v_prototype` (one per LoRA module, ~144 for Qwen3-4B q/k/v/o)
- At inference time on the user prompt:
  - Forward pass through model with hooks on every LoRA-wrapped Linear
  - For each module: pool input as **mean over all tokens**, compute `cos(pooled, v_prototype)`
  - Aggregate across all 144 modules: take **MAX of |cos|** across modules
  - Pass through sigmoid: `α(x) = sigmoid((max|cos| − bias) / T)`, calibrated `bias=0.158, T=0.02`
- Use `α` as the LoRA mixing weight in the FedALT mixer (Layer 3)

**Counter-intuitive findings from Phase 0 smoke testing:**
- Mean cosine across modules **fails** (dilutes signal). MAX is required.
- Last-token pooling is **worse** than mean pooling on this LoRA (the LoRA fires throughout the prompt, not just at the decision token).
- Absolute cosine `|cos|` is required because the LoRA can shift hidden states in either direction along the prototype.

**Cost:** ~free at inference (one forward pass on the prompt before generation; could fold into the first generation step itself).
**Reported (paper):** "Nearly matches Oracle routing" on 256 upstream tasks; +1.8 pp on Phi-2, +2.5 pp on Mistral.
**Measured (Phase 0 ckpt-1600, N=30 per set):** 95% balanced accuracy, margin Python − OOD ≈ +0.08.
**Source:** [Arrow Routing — Ostapenko et al., NeurIPS 2024](https://arxiv.org/html/2405.11157v1)

**Risk note (resolved):** Arrow was validated in the paper on multi-LoRA libraries (10s of LoRAs). Our case is "1 personal LoRA + 1 backbone". I rated the risk at ~20-30% pre-experiment. **Phase 0 smoke real result: works**, even on a partially-trained adapter. Final eval on the fully-trained adapter will confirm.

**Fallback if Arrow underperforms on production LoRAs:** SpectR (full spectrum extension of Arrow, COLM 2025) or LoraRetriever with our existing MiniLM.

---

### Layer 3 — FedALT-Style Mixer

**Purpose:** Blend personal LoRA output with server backbone output, per-token, weighted by Arrow score.

**Implementation:**
```
output(x) = W₀x + α(x) · B^L A^L x + (1 - α(x)) · server_output(x)
```
where:
- `W₀` = backbone weights (frozen, e.g., Qwen 3.5 4B)
- `B^L A^L` = personal LoRA adapter (~50 MB)
- `α(x)` = mixing weight from Layer 2 (Arrow score)
- `server_output(x)` = parallel inference from a larger server backbone

**Cost:** Standard LoRA inference + one extra forward pass through server backbone
**Source:** [FedALT — Federated Adaptive Local Training for LoRA, 2025](https://arxiv.org/html/2503.11880)
**Production reference:** Apple Intelligence ships ~160 MB LoRA adapters at 3.7 bpw on a 3B on-device foundation model — our 50 MB target is realistic with aggressive quantization.

---

### Layer 4 — Self-REF Confidence Token

**Purpose:** After generation, the personal LoRA reports "am I confident?" via a special logit.

**Implementation:**
- Add `[CONF]` token to vocabulary during LoRA SFT
- Training data construction:
  - For each Q&A pair, run inference with personal LoRA
  - If answer matches ground truth → label `[CONF=high]`
  - If wrong → label `[CONF=low]`
- At inference: read the `[CONF]` logit at end of generation
- If `CONF < threshold` → escalate to Layer 5

**Cost:** +1 token per query (negligible); +5-10% to LoRA training time
**Why this beats verbalized confidence:** Asking an LLM "rate yourself 1-10" is unreliable — Llama-3.3-70B reports 90-100% confidence when actual accuracy is ~40%. A trained `[CONF]` token learns to correlate with **factual correctness**, not "sounding confident".
**Source:** [Self-REF / Confidence Tokens — ICML 2025](https://arxiv.org/html/2410.13284v3)

---

### Layer 5 — Cascade Escalation

**Purpose:** If Self-REF reports low confidence, fall back to pure server backbone.

**Implementation:**
- If `[CONF] < threshold` → discard personal LoRA output
- Re-run query through server backbone alone (no LoRA mixing)
- Optional: AutoMix-style POMDP meta-verifier smooths noisy signals

**Cost:** Only when escalation triggers (estimated 10-30% of queries early on, decreasing as personal LoRA improves)
**Source:** [AutoMix — Madaan et al., NeurIPS 2024](https://arxiv.org/abs/2310.12963)

---

## Model Ladder (corrected post-Phase-0)

**Jippy = phone-side LoRA (always)**. The ladder is the SERVER backbone that pairs with Jippy at each user level.

| Level | Phone (offline mode) | Server backbone (online mode) | Hardware | Cost/hr |
|-------|---------------------|-------------------------------|----------|---------|
| 1-2   | Jippy LoRA + SmolLM 1.7B / Phi-3 mini | Jippy LoRA + Qwen 3.5 4B | RTX 3060 8GB | $0.15 |
| 3-4   | (same)              | Jippy LoRA + Qwen 3.5 9B / Mistral Nemo | RTX 3090 12GB | $0.30 |
| 5-6   | (same)              | Jippy LoRA + Qwen 3.5 27B / Gemma 4 31B | RTX 4090 24GB | $0.50 |
| 7+    | (same)              | Jippy LoRA + Llama 3.3 70B / Qwen 2.5 72B | A100 80GB | $1.00 |

**Two synchronized LoRAs:** since LoRA weights live in the host model's hidden-state space, we actually need TWO LoRAs per user — one for the offline (SmolLM) backbone and one for whatever server backbone the user is currently paired with. They're trained from the same user data via parallel SFT runs.

**Backbone update policy:** the server backbone is only swapped when a real game-changer ships (not on every minor release). When swapped, the user's server-side LoRA is re-trained on the new backbone (one-time job, ~5 min on vast.ai). Phone LoRA is unchanged.

**Endgame (post-MVP):** Final boss fights against **ChatGPT 5.4** and **Claude Opus 4.6** via paid APIs. Cross-judge pattern: Claude judges GPT fights, GPT judges Claude fights. Reserved for users who beat lvl 7. APIs are also used as Layer 5 cascade fallback for low-confidence queries.

---

## Knowledge Bank & XP System

### Ingest Pipeline
```
User feeds [text / URL / voice]
   ↓
[Whisper Tiny] for voice → text (75 MB, CPU)
   ↓
[DistilBERT-multilingual] quality filter (130 MB, CPU)
  - garbage / spam → 0 XP
   ↓
Chunking: 200-token chunks
   ↓
[MiniLM-L6-v2] embedding per chunk (25 MB, CPU)
   ↓
Dedup vs existing KB:
  - max_cosine > 0.85 → DUPLICATE → +1 XP
  - 0.65 < max_cosine < 0.85 → PARTIAL → +20 XP
  - max_cosine < 0.65 → NEW FACT → +100 XP
   ↓
Specialization bonus:
  - 80%+ in same topic cluster → +50% XP
  - 100% in same topic → +75% XP
   ↓
Store chunk + embedding in knowledge_bank (Postgres + pgvector)
```

### XP Curve
```
xp_to_next_level(lvl) = round(100 * lvl^1.5)
```
| Lvl | XP needed |
|-----|-----------|
| 2   | 283       |
| 3   | 520       |
| 4   | 800       |
| 5   | 1118      |
| 6   | 1470      |
| 7   | 1852      |

### Anti-Farm Protections
- Daily XP cap: 1000 XP/day
- Cooldown between submissions: 30 seconds
- Quality filter on ingest (DistilBERT)
- **No diversity bonus** — we want narrow specialists, not generalists

### Level-Up Trigger
1. User feeds material → XP accumulates
2. When `total_xp >= xp_to_next_level(current+1)` → trigger LoRA training on server (~5 min on vast.ai)
3. Server backbone auto-generates Q&A pairs from new chunks, used as training data
4. New LoRA weights pushed to phone
5. Level-up animation + boss fight unlock

---

## Game Design (MVP — minimal)

| Lvl | Unlock |
|-----|--------|
| 1 | Pasture (feed), Tavern (give name) |
| 2 | First boss fight |
| 3 | Blacksmith (deepen / widen architecture) |
| 4 | Boss fight |
| 5 | Mage (mystical experimental upgrades) |
| 6+ | Boss fights, repeat upgrade cycle |

Cosmetics, voice chat, multi-Jippy, marketplace, PvP — all post-MVP.

---

## What We're Testing — Phase 0 Experiment

**Goal:** Validate Arrow Routing and Self-REF on our exact use case (1 narrow LoRA + 1 general backbone) before committing to production.

**Setup:**
- Hardware: vast.ai RTX 3090 24GB ($0.30/hr × ~10 hrs = ~$3)
- Base model: Qwen 3.5 4B
- Personal LoRA: fine-tuned on CodeAlpaca-Python (~50K Q&A pairs, ~2 hrs training)
- Eval set: 1000 queries — 500 in-domain (Python), 500 out-of-domain (MMLU mixed)

**Variants:**
| ID | Configuration |
|----|---------------|
| A  | Baseline: Qwen 3.5 4B alone (no LoRA) |
| B  | Always-on LoRA: LoRA always active with fixed weight 0.3 (Apple-style) |
| C  | **Arrow Routing**: SVD prototype + sigmoid mixing |
| D  | **SpectR**: full spectrum routing |
| E  | **Arrow + Self-REF**: Variant C plus `[CONF]` token training |

**Metrics:**
- In-domain accuracy (Python — code execution + BLEU)
- Out-of-domain perplexity (MMLU)
- Routing accuracy (does the gate correctly activate / deactivate?)
- Speed overhead per query

**Decision rules:**
- If `C > A by ≥3%` and OOD doesn't degrade → Arrow ships in production
- If `D > C by ≥2%` → upgrade to SpectR
- If `E > C by ≥5%` → ship Arrow + Self-REF combo
- If everything is ≤ Always-on (B) → ship Apple-style always-on, drop Arrow

**Timeline:** 2 days of work + ~10 hrs of GPU. Total cost ~$3-5.

---

## Implementation Phases

| Phase | Description | Time | Cost |
|-------|-------------|------|------|
| **0** | Phase 0 experiment — validate Arrow + Self-REF on vast.ai | 2 days | $5 |
| **1** | Spin up Ollama on vast.ai + Cloudflare tunnel + Fastify integration | 1 day | — |
| **2** | Ingest pipeline (Whisper + DistilBERT + MiniLM + dedup XP scoring) | 2 days | — |
| **3** | LoRA training trigger on XP threshold; auto-generate Q&A from chunks | 1 day | — |
| **4** | End-to-end test on iPhone with one user | 1 day | — |
| **5** | Implement Arrow routing in inference pipeline (or fallback) | 1 day | — |
| **6** | Add RAG layer (knowledge_bank embeddings → context injection) | 1 day | — |
| **7** | First boss fight (lvl 2) — battle UI + arena flow | 2 days | — |
| **8** | Quality validation + production polish | 1 day | — |

**Total: ~12-14 days + ~$5-10 for vast.ai experiments + ~$220/month for RTX 3090 server in production.**

---

## What We Use vs Avoid

### Use (validated for our case)
- **Arrow Routing** (NeurIPS 2024) — primary routing
- **Self-REF Confidence Tokens** (ICML 2025) — confidence signal
- **FedALT mixer** (2025) — output blending
- **MiniLM-L6-v2** — embeddings for RAG and dedup
- **DistilBERT-multilingual** — quality filter
- **Whisper Tiny** — voice ingestion
- **Apple Intelligence quantization recipe** — 3.7 bpw for personal LoRA

### Fallback (if primary fails)
- **SpectR** (COLM 2025) — full spectrum routing if Arrow's top-1 SVD vector is too coarse
- **LoraRetriever** (ACL 2024) — embedding-based routing using existing MiniLM
- **Always-on LoRA** (Apple-style) — no routing at all, fixed weight

### Avoid (V5 failure mode)
- **AdapterFusion** — learned attention gate, this is what V5 was
- **MoLE / MixLoRA / LD-MoLE** — assume joint training of experts and router from scratch
- **LoRAHub** — needs few-shot examples per query, too slow
- **Verbalized confidence** ("rate yourself 1-10") — proven unreliable, 90% overconfidence
- **Speculative decoding** — saves latency not cost, doesn't fit our goal

---

## Citations (key papers)

### Routing & Adapter Composition
- [Arrow Routing — Ostapenko et al., NeurIPS 2024](https://arxiv.org/html/2405.11157v1)
- [SpectR — COLM 2025](https://arxiv.org/abs/2504.03454)
- [LoraRetriever — Zhao et al., ACL 2024](https://arxiv.org/abs/2402.09997)
- [LoRAuter — arxiv 2026](https://arxiv.org/html/2601.21795v1)
- [LAG: LoRA-Augmented Generation — 2025](https://arxiv.org/abs/2507.05346)
- [MoErging Survey — Yadav et al.](https://arxiv.org/pdf/2408.07057)

### Confidence & Cascading
- [Self-REF Confidence Tokens — ICML 2025](https://arxiv.org/html/2410.13284v3)
- [AutoMix — NeurIPS 2024](https://arxiv.org/abs/2310.12963)
- [FrugalGPT — Chen et al., 2023](https://arxiv.org/abs/2305.05176)
- [Cascade Routing — ICLR 2025](https://files.sri.inf.ethz.ch/website/papers/dekoninck2024cascaderouting.pdf)

### Personalized LLMs
- [FedALT — 2025](https://arxiv.org/html/2503.11880)
- [Apple Foundation Models Tech Report 2025](https://machinelearning.apple.com/research/apple-foundation-models-tech-report-2025)
- [LaMP Benchmark — ACL 2024](https://lamp-benchmark.github.io/)
- [MobiLoRA — ACL 2025](https://aclanthology.org/2025.acl-long.1140/)

### Models
- [GLM-5.1 — Z.ai April 2026](https://huggingface.co/zai-org/GLM-5)
- [Qwen 3.5 family](https://qwenlm.github.io/)
- [Llama 3.3 70B](https://huggingface.co/meta-llama/Llama-3.3-70B-Instruct)

---

## Open Questions / Risks

1. **Arrow Routing on single LoRA setup** — needs Phase 0 validation (~20-30% risk of silent failure)
2. **CONF token training data** — how many Q&A pairs needed before CONF logit becomes reliable? Hypothesis: ~500. Will measure in Phase 0E.
3. **LoRA training quality at low XP** — at lvl 2 (283 XP, ~3 chunks), is there enough signal for meaningful LoRA fine-tune? Phase 4 experiment will validate.
4. **Phone-side ONNX Runtime** — currently configured for V3 architecture. Will need updates for new LoRA mixing logic.
5. **Endgame API costs** — Claude Opus 4.6 and ChatGPT 5.4 API pricing at scale needs budgeting; cross-judge requires 2 paid API calls per endgame fight.

---

## Decision Log

| Date | Decision | Rationale |
|------|----------|-----------|
| 2026-04-08 | V5 fusion routing failed (Track 0 wins 93.2%) | Donors hurt instead of help; proj_in/proj_out never learned |
| 2026-04-11 | Switch to Arrow Routing + Self-REF + FedALT mixer | Validated in literature 2024-2026; avoids learned gate failure mode |
| 2026-04-11 | Drop diversity bonus, add specialization bonus | Product vision is narrow specialists |
| 2026-04-11 | Remove lvl 8 endgame from MVP | GLM-5.1 / Kimi K2.5 don't fit single A100 |
| 2026-04-11 | Phase 0 result: Arrow Routing validated at 95% bal acc | smoke real on N=30, margin Python − OOD ≈ +0.08 |
| 2026-04-11 | Loss is misleading on OOD eval | Always-on LoRA wins by loss but generation jaccard shows it's format mimicry, not knowledge transfer |
| 2026-04-11 | Add Layer 0 "Format LoRA" always-on | Stop pretending specialist LoRA helps OOD; let a dedicated tiny adapter own format-following |
| 2026-04-11 | Jippy = phone-side LoRA (always) | "Level up" is server backbone, not Jippy; LoRA travels with user, two synchronized LoRAs per user (phone + server backbone family) |
| 2026-04-11 | Use MAX_NEW≥1024 on reasoning/math/code generation evals | At MAX_NEW=128, GSM8K solutions get cut off mid-solve, can't compare reasoning quality |
| 2026-04-12 | Phase 1 (Hole #4): train Math LoRA + multi-LoRA composition | Need to confirm Arrow correctly weights two specialists per query; foundation for guild routing at scale |
| 2026-04-11 | Phase 0 experiment mandatory before production | Avoid second V5-style silent failure |
