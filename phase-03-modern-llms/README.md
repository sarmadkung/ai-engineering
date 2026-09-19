# Phase 3 — Modern LLMs

**Goal:** understand the difference between your tiny model and a frontier one — architecturally,
economically, and behaviourally. You'll modernize your own model, and run a real fine-tune and
alignment pass.

---

## What I will do

| # | Topic | What I build | Status |
|---|---|---|---|
| 11 | [Transformer Deep Dive](11-transformer-deep-dive/) | `variants.py` — encoder/decoder/cross-attention comparisons | ready |
| 12 | [Modern LLM Architecture](12-modern-llm-architecture/) | `modern_model.py` — RMSNorm, RoPE, SwiGLU, GQA, MoE | ready |
| 13 | [LLM Training](13-llm-training/) | `training_math.py`, `pipeline.py` — cost models and a real data pipeline | ready |
| 14 | [Fine-Tuning](14-fine-tuning/) | `sft.py`, `lora.py` — a genuinely fine-tuned model | ready |
| 15 | [Alignment](15-alignment/) | `reward_model.py`, `dpo.py` — preference training | ready |

## Which things to learn

**11. Transformer Deep Dive** — what the transformer removed from RNNs and why that mattered for
training; encoder-only, decoder-only and encoder-decoder; cross-attention; why decoder-only won;
pre-norm; what changed between 2017 and now and what didn't.

**12. Modern LLM Architecture** — RMSNorm; RoPE and relative position (and why it extends context when
learned embeddings can't); SwiGLU's gate; the KV cache and its memory arithmetic; grouped-query
attention; Mixture of Experts, and total versus active parameters.

**13. LLM Training** — what pretraining produces; the data pipeline (extraction, quality filtering,
deduplication, decontamination, mixture); data, tensor and pipeline parallelism; ZeRO/FSDP; the
`6ND` FLOPs formula; scaling laws, Chinchilla, and why labs now over-train small models.

**14. Fine-Tuning** — instruction tuning and what it actually changes; SFT with prompt masking; chat
templates as a contract; LoRA and why low rank suffices; QLoRA; catastrophic forgetting; and — the most
valuable judgement here — when *not* to fine-tune.

**15. Alignment** — why SFT can't teach quality; preference data and annotator disagreement; reward
models; RLHF and reward hacking; DPO and what it removes; safety alignment and the
helpfulness/harmlessness trade; why jailbreaks work at all.

## Prerequisites

Phase 2 complete. For Topics 14–15 you need `transformers`, `peft`, `trl`, `bitsandbytes` and a small
base model (0.5B–1B) — a laptop GPU or a cheap rented one is enough.

**Next:** Phase 4 — you stop building models and start building products with them.
