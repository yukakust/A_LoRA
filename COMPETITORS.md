# gpu.social -- Competitive Landscape & Potential Allies

Last updated: 2026-04-02

gpu.social differentiator: uses **consumer phones/devices** for distributed GPU computing (mobile-first DePIN). Most competitors focus on data-center GPUs or desktop gaming rigs.

---

## 🔭 СЛЕДИМ (Open Source, Ближайшие аналоги)

### mesh-llm (AnarchAI)

- **Docs:** https://docs.anarchai.org
- **GitHub:** https://github.com/michaelneale/mesh-llm ← **СЛЕДИМ за репо**
- **Автор:** Michael Neale
- **Статус:** WIP, активная разработка. MIT лицензия.
- **Стек:** Rust + iroh (QUIC) + llama.cpp. Нет экономики (volunteer).
- **Целевые устройства:** Десктоп GPU (Mac, Linux). Мобильное приложение запланировано ("AirDrop for AI").

**Что они решили и как (изучено):**

| Проблема | Их решение | Наш статус |
|----------|-----------|------------|
| Expert sharding | gate_mass ranking → shared core (топ N) реплицирован на всех, остальные round-robin+overlap | ✅ Берём |
| Zero cross-node traffic | Каждая нода = полный trunk + subset экспертов → независимый llama-server | ✅ Наша архитектура |
| Node discovery | Nostr Kind 31990 события + auto-join scoring | V4 план |
| Demand propagation | Gossip demand map, merge via max(), 24h TTL, 60s standby-promote | V4 план |
| P2P protocol | QUIC через iroh (мультиплексинг gossip/RPC/HTTP) | V4 план |
| Speculative decoding | +38% throughput, 75% acceptance | V3 план |
| Zero-transfer loading | SET_TENSOR_GGUF: нода читает веса с локального диска (111s→5s init, 558ms→8ms per-token) | ✅ Наша архитектура |

**Чего у них нет:**
- Экономики (Compute Miles) — у нас есть
- Обучения модели — они берут готовые GGUF
- Мобильного inference — только десктоп пока
- Персонализации (per-user expert delta) — у нас есть

**Главный вывод:** Их `moe.rs` и `mesh.rs` — готовый reference implementation для V4. Читать код как учебник.

---

## 1. Akash Network

- **URL:** https://akash.network
- **Founder:** Greg Osuri
- **Twitter:** [@gregosuri](https://x.com/gregosuri)
- **Status (Q4 2025):** Live mainnet. New leases up 28% QoQ (34,300 in Q4 2025). Launched AkashML managed inference layer. Mainnet 14 upgrade migrated to Cosmos SDK v0.53.
- **Funding:** Starcluster initiative -- up to $75M via regulated "Starbonds" to fund ~7,200 NVIDIA GB200 GPUs operated by enterprise-grade datacenter "Nodekeepers."
- **Technical approach:** Decentralized cloud marketplace on Cosmos SDK. Kubernetes-based deployments. New Burn-Mint Equilibrium (BME) tokenomics launching March 2026. **Akash Homenode** (Feb 2026) now allows consumer-grade GPUs to contribute compute -- this directly overlaps with gpu.social's territory.
- **Difference from gpu.social:** Akash targets server/datacenter hardware primarily; Homenode is new and desktop-focused, not mobile. Uses Kubernetes containers, not lightweight mobile inference.
- **Ally or competitor:** COMPETITOR. Homenode is a direct move into consumer hardware. However, Akash's marketplace could be an integration target -- gpu.social devices could appear as Akash providers.

---

## 2. Render Network

- **URL:** https://rendernetwork.com
- **Founder:** Jules Urbach (CEO of OTOY)
- **Twitter:** [@JulesUrbach](https://x.com/JulesUrbach)
- **Status:** Live. 63M cumulative frames rendered. 40% compute power increase in 2025. 35% of output from Hollywood studios and AI training clients. RenderLabs spun out as for-profit entity in 2025.
- **Funding:** Token-funded (RENDER). Significant market cap. Backed by strong Hollywood/creative industry partnerships.
- **Technical approach:** Decentralized GPU rendering for 3D/VFX content. Dispersed AI subnet (Dec 2025) aggregates decentralized GPUs for enterprise-grade AI workloads (H100/H200, AMD MI300X). Originally focused on creative rendering, now expanding into AI inference.
- **Difference from gpu.social:** Render targets desktop GPUs (gaming rigs, creative workstations) for rendering and AI. Not mobile. Focuses on high-end GPU tasks, not lightweight on-device inference.
- **Ally or competitor:** POTENTIAL ALLY. Render serves creative/rendering workloads; gpu.social serves mobile inference. Complementary if gpu.social handles edge inference while Render handles heavy rendering.

---

## 3. Bittensor

- **URL:** https://bittensor.com
- **Founder:** Jacob Robert Steeves (co-founder; "Yuma Rao" is pseudonymous co-author)
- **Twitter:** [@const_reborn](https://x.com/const_reborn)
- **Status:** 128 active subnets (97% growth in 2025). $620M total value staked. First halving Dec 2025 cut emissions 50%. TAO rallied 102% in one month (March 2026). Grayscale and Bitwise filed for spot TAO ETFs.
- **Funding:** Token-funded (TAO). ~$1.37B subnet valuation, though sustained by ~$52M/year in TAO subsidies rather than organic revenue.
- **Technical approach:** Incentivized network of AI subnets. Each subnet specializes (LLM training, inference, data, etc.). Covenant-72B was trained by 70+ independent contributors using everyday GPUs over ordinary internet -- proving decentralized LLM pre-training works. NVIDIA CEO publicly recognized achievement.
- **Difference from gpu.social:** Bittensor is a marketplace/incentive layer for AI models, not a raw compute network. Subnets are high-level AI tasks. Not mobile-focused at all -- contributors use desktop/server GPUs.
- **Ally or competitor:** POTENTIAL ALLY. gpu.social could operate as a Bittensor subnet providing mobile-edge inference capacity. Bittensor's subnet model is designed for exactly this kind of integration.

---

## 4. io.net

- **URL:** https://io.net
- **Founded by:** Ahmad Shadid (stepped down June 2024). Current CEO: Gaurav Sharma. Foundation Chair: Tory Green.
- **Twitter:** [@sharmag88](https://x.com/sharmag88) (Gaurav Sharma)
- **Status:** Live on Solana. Launched Co-Staking Marketplace (Feb 2025) and Incentive Dynamic Engine (Dec 2025). Claims services up to 90% cheaper than traditional cloud.
- **Funding:** Raised $40M Series A (a16z crypto, Hack VC, others). Solana-based $IO token.
- **Technical approach:** Aggregates idle GPU/CPU resources into a DePIN network on Solana. Focuses on ML/AI workloads. Uses idle computing power from data centers, crypto miners, and individual contributors.
- **Difference from gpu.social:** io.net focuses on aggregating data-center and miner GPUs, not mobile devices. Desktop/server oriented. Solana-based like Nosana.
- **Ally or competitor:** COMPETITOR. Direct overlap in "aggregate idle compute for AI" narrative. But io.net doesn't do mobile -- if gpu.social nails mobile inference, io.net could be a partnership for hybrid workloads (mobile edge + server backend).

---

## 5. Gensyn

- **URL:** https://gensyn.ai
- **Founders:** Harry Grieve (CEO), Ben Fielding (CTO)
- **Twitter:** [@_grieve](https://x.com/_grieve)
- **Status:** Public testnet launched March 2025. $AI token presale Dec 2025 via Sonar platform.
- **Funding:** $80.6M total. $43M Series A (June 2023) led by a16z Crypto.
- **Technical approach:** Decentralized protocol for ML training verification. Key innovation: cryptographic verification of training work done on untrusted hardware. Aims to unify compute from personal computers to data centers. Focuses on training (not inference).
- **Difference from gpu.social:** Gensyn focuses on training verification, not inference. Desktop/server hardware. Their verification protocol is interesting -- could be applied to mobile compute to prove work was done correctly.
- **Ally or competitor:** STRONG POTENTIAL ALLY. Gensyn's verification layer could validate that gpu.social mobile devices actually performed the inference correctly. Complementary: Gensyn = training verification, gpu.social = mobile inference execution.

---

## 6. Nosana

- **URL:** https://nosana.com
- **Founder:** Jesse Eisses
- **Twitter:** [@JesseEisses](https://x.com/JesseEisses)
- **Status:** Mainnet launched Jan 2025. 50,000+ GPU hosts. 2M+ deployments by August 2025. Solana-based.
- **Funding:** Token-funded (NOS on Solana). Claims 85% lower costs than centralized GPU providers.
- **Technical approach:** Decentralized GPU marketplace on Solana focused on AI inference. Community-contributed consumer GPUs (desktop). Grants program for ecosystem tooling.
- **Difference from gpu.social:** Nosana targets desktop consumer GPUs, not mobile. Similar vision (consumer hardware for AI inference) but different device class. Solana-based.
- **Ally or competitor:** DIRECT COMPETITOR in vision (consumer hardware for AI inference). However, Nosana = desktop GPUs, gpu.social = mobile GPUs. Could coexist or partner to cover both segments.

---

## 7. Together AI

- **URL:** https://together.ai
- **Founder:** Vipul Ved Prakash (CEO)
- **Twitter:** [@vipulved](https://x.com/vipulved)
- **Status:** $300M annualized revenue (Sep 2025). Multiple data centers (Maryland, Memphis, Sweden). Deploying 36,000 NVIDIA GB200 NVL72 GPUs with Hypertec.
- **Funding:** $305M Series B (Feb 2026) at $3.3B valuation. Led by General Catalyst. NVIDIA participated.
- **Technical approach:** Centralized AI cloud optimized for open-source models. NOT decentralized -- they own/operate data centers. Focus on inference speed and cost for enterprise customers. 200 MW of power capacity.
- **Difference from gpu.social:** Together AI is a centralized cloud provider, the opposite of decentralized. Massive capital, enterprise focus. They compete with AWS/Azure, not with DePIN projects.
- **Ally or competitor:** INDIRECT COMPETITOR / BENCHMARK. Together AI shows what centralized scale looks like ($3.3B valuation). gpu.social competes on cost and decentralization, not on raw performance. Together AI's pricing is the benchmark gpu.social must beat.

---

## 8. Theta Network

- **URL:** https://thetatoken.org
- **Founder:** Mitch Liu (CEO)
- **Twitter:** [@mitchliu](https://x.com/mitchliu)
- **Status:** EdgeCloud Hybrid launched June 2025. 30,000+ community edge nodes. Partnerships with Deutsche Telekom, NTT Digital, Imperial College London.
- **Funding:** Token-funded (THETA, TFUEL). Significant market cap.
- **Technical approach:** Originally a decentralized video CDN. Pivoted to EdgeCloud for AI: combines cloud GPUs with 30,000+ distributed edge nodes for inference and training. Distributed inferencing splits large LLMs across multiple edge nodes. Enterprise-focused partnerships.
- **Difference from gpu.social:** Theta has 30K edge nodes but primarily on desktops/servers, not mobile. Strong enterprise partnerships that gpu.social lacks. Theta's distributed inference (splitting LLMs across nodes) is technically relevant to what gpu.social wants to do.
- **Ally or competitor:** COMPETITOR with lessons to learn. Theta's EdgeCloud Hybrid architecture (cloud + edge) is a model for what gpu.social could become. Their telecom partnerships (Deutsche Telekom, NTT) show a go-to-market path.

---

## 9. Flux

- **URL:** https://runonflux.com
- **Founder:** Daniel Keller
- **Twitter:** [@dak_flux](https://x.com/dak_flux)
- **Status:** 10,000-12,000 nodes across 67 countries, 560+ independent operators. 25,000+ applications deployed. Transitioned to Proof-of-Useful-Work v2 (Oct 2025).
- **Funding:** Token-funded (FLUX). Self-sustaining through node economics.
- **Technical approach:** Decentralized cloud infrastructure (containers, storage, networking). Proof-of-Useful-Work means nodes must run actual AI/app workloads to earn rewards. General-purpose Web3 cloud, not AI-specific.
- **Difference from gpu.social:** Flux is general-purpose cloud (containers, apps), not focused on AI inference. Server/desktop nodes, not mobile. More comparable to a decentralized AWS than a mobile compute network.
- **Ally or competitor:** LOW OVERLAP. Flux could host gpu.social's coordination/orchestration layer. Not competing for the same compute providers (servers vs phones).

---

## 10. iExec

- **URL:** https://iex.ec
- **Founder:** Gilles Fedak
- **Twitter:** [@gilfedak](https://x.com/gilfedak)
- **Status:** Live. Pivoted toward privacy-focused compute with Trusted Execution Environments (Intel SGX). All 87M RLC tokens in circulation (no dilution risk).
- **Funding:** ICO-funded. Established since 2016.
- **Technical approach:** Decentralized marketplace for compute, data, and apps. Key differentiator: TEE-based confidential computing (Intel SGX enclaves). Proof-of-Contribution consensus. Focus on data privacy -- users can monetize data without exposing it.
- **Difference from gpu.social:** iExec focuses on confidential computing and data privacy, not raw GPU inference. Desktop/server hardware. Their TEE approach is interesting for privacy-sensitive inference.
- **Ally or competitor:** POTENTIAL TECHNOLOGY PARTNER. iExec's confidential computing techniques could be valuable if gpu.social needs to guarantee privacy on untrusted mobile devices.

---

## Emerging / Notable Mentions

### Destra Network
- Building a mobile app (Destra Edge) to let smartphone GPUs contribute to decentralized AI inference. Demo shown Jan 2026, public app planned Q1 2026.
- **CLOSEST DIRECT COMPETITOR** to gpu.social's mobile-first thesis. Watch closely.

### 0G Labs
- Trained DiLoCoX-107B (107B params) across distributed nodes on ordinary 1 Gbps internet. 357x better communication efficiency than standard AllReduce. Collaboration with China Mobile.
- Relevant because their distributed training tech works on consumer internet speeds.

### Aethir
- Delivered 1.4B compute hours, ~$40M quarterly revenue in 2025. Enterprise GPU cloud with decentralized elements.

---

## Summary Matrix

| Project | Mobile? | Focus | Threat Level | Best Relationship |
|---------|---------|-------|-------------|-------------------|
| Akash | No (Homenode = desktop) | General cloud | HIGH | Integration target |
| Render | No (desktop GPUs) | Rendering + AI | LOW | Complementary ally |
| Bittensor | No (desktop/server) | AI model marketplace | LOW | Subnet integration |
| io.net | No (data center/miner) | GPU aggregation | MEDIUM | Hybrid partner |
| Gensyn | No (desktop/server) | Training verification | LOW | Verification layer |
| Nosana | No (desktop GPUs) | AI inference | MEDIUM | Co-marketing |
| Together AI | No (own data centers) | Centralized AI cloud | LOW (different market) | Benchmark |
| Theta | No (desktop edge nodes) | Edge AI + video | MEDIUM | Architecture model |
| Flux | No (servers) | General cloud | LOW | Infra hosting |
| iExec | No (desktop/server) | Confidential compute | LOW | Privacy tech partner |
| **Destra** | **YES (mobile)** | Mobile GPU inference | **HIGHEST** | Direct competitor |

**Key insight:** Almost nobody is doing mobile-first distributed GPU compute. gpu.social's phone-based approach is genuinely differentiated. Destra Network is the only direct mobile competitor identified. The main risk is that desktop-focused projects (Akash Homenode, Nosana) expand downward to mobile before gpu.social scales up.
