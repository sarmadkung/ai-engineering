# Phase 11 — Local & Open-Source LLMs

**Goal:** run and serve models yourself, and be able to defend the local-versus-API decision with
numbers rather than preference.

---

## What I will do

| # | Topic | What I build | Status |
|---|---|---|---|
| 44 | [Open-Source Models](44-open-source-models/) | `model_selection/` — evaluation-driven model choice and a break-even model | ready |
| 45 | [Local Inference](45-local-inference/) | `benchmarks/` — quantization, speed, and quality sweeps | ready |
| 46 | [Model Serving](46-model-serving/) | `serving/` — an inference service with load testing | ready |

## Which things to learn

**44. Open-Source Models** — open weights versus open source, and licences that restrict commercial use;
the families (Llama, Qwen, Mistral, Gemma, DeepSeek, Phi) and their characters; base versus instruct
versus reasoning variants; memory arithmetic; what small models are genuinely good at and where they fail;
the real reasons to go local and the real costs; **selection by your own evaluation set, not
leaderboards**; the cost break-even including engineering time.

**45. Local Inference** — Ollama, llama.cpp, vLLM and Transformers, and what each is for; quantization
formats and the 4-bit sweet spot; the heuristic that a bigger model at 4-bit usually beats a smaller one
at 16-bit; **why generation speed tracks memory bandwidth**; prefill versus decode; batching; context
length costs; chat-template mismatch as a silent quality killer.

**46. Model Serving** — OpenAI-compatible APIs and where compatibility ends; **continuous batching**
versus static; PagedAttention and prefix caching; throughput versus latency as a decision you make;
diagnosing low GPU utilization with a queue (not a hardware problem); horizontal scaling and
prefix-aware routing; LoRA multi-tenancy; and why **utilization is your unit economics** when
self-hosting.

## Prerequisites

Ollama or llama.cpp (any machine). vLLM needs an NVIDIA GPU — where you can't run something, work the
arithmetic and say so in your notes. Your Phase 9 evaluation set is the instrument for every quality
claim in this phase.

**Next:** Phase 12 — the production infrastructure around all of it.
