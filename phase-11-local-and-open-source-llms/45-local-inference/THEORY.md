# Topic 45 — Local Inference

**Why this topic:** running a model yourself. The tools are easy to start with and full of decisions
that determine whether you get 5 or 50 tokens per second. This is also where Phase 2's inference
knowledge becomes operational.

---

# Part 1 — Theory

## 45.1 The tool landscape

Four things people mean by "running a model locally", with different purposes:

**Ollama** — the easy path. `ollama run qwen3` and you have a model with an OpenAI-compatible HTTP
API. Handles downloads, quantization selection, and model swapping. Built on llama.cpp. Best for
development, single-user use, and getting started. Not built for serving many concurrent users.

**llama.cpp** — the C++ engine underneath much of the ecosystem. Runs on CPU, GPU, or both (offloading
some layers), on almost any hardware, with GGUF quantized files. Maximum portability and control;
lower-level. Its `llama-server` is a real server with continuous batching.

**vLLM** — the production serving engine. GPU-only, built around **PagedAttention** (efficient KV
cache memory management) and **continuous batching**. Very high throughput under concurrency, an
OpenAI-compatible API, and support for tensor parallelism across GPUs. The right answer for serving
(Topic 46). Alternatives in the same class: SGLang, TensorRT-LLM.

**Transformers (Hugging Face)** — the research/flexibility path. Full control, easy to inspect and
modify, slow for serving. What you'd use to understand or experiment.

Choosing: Ollama to develop, llama.cpp for CPU/edge/portability, vLLM to serve.

## 45.2 Quantization

Storing weights in fewer bits. The core trade of local inference.

```
fp16     16 bits   baseline quality        ~2.0 GB per 1B params
int8      8 bits   near-identical          ~1.0 GB
int4      4 bits   small but real loss      ~0.5 GB
int2-3   2-3 bits  significant loss
```

Formats you'll meet: **GGUF** (llama.cpp/Ollama, CPU+GPU, many variants like `Q4_K_M`), **GPTQ** and
**AWQ** (GPU, calibrated post-training quantization), **bitsandbytes** (on-the-fly, used in QLoRA —
Topic 14).

What actually matters in practice:

- **4-bit is the usual sweet spot** — the quality loss is modest and the memory saving is large.
- **A bigger model at 4-bit generally beats a smaller model at 16-bit** for the same memory. This is
  the most useful heuristic in the topic.
- Quality loss is **task-dependent**: it shows up first in reasoning and precise formatting, and
  barely at all in classification or summarization. So measure on *your* task rather than trusting a
  perplexity number.
- Below 4-bit, degradation accelerates.

## 45.3 GPU vs CPU, and what limits speed

Generation is **memory-bandwidth bound**, not compute bound: each token requires reading the model's
weights from memory. That single fact explains most of local inference performance.

```
tokens/second ≈ memory bandwidth / model size in memory
```

Consequences:

- A GPU is fast mainly because its memory bandwidth is ~10–20× a CPU's.
- Apple Silicon does well because unified memory has high bandwidth and the GPU can use all of it.
- **Quantization speeds up generation**, not just memory use, because there are fewer bytes to read.
- **Prefill** (processing the prompt) *is* compute-bound and parallel, which is why long prompts are
  cheap per token and generation is not (Topic 10).
- **Partial offloading** (some layers on GPU, rest on CPU) is dominated by the slowest path; performance
  falls off sharply once you spill.

## 45.4 Practical speed factors

- **Batching** — generating for several requests at once costs barely more than one, because the
  weight read is shared. This is why serving throughput is so much higher than single-stream speed
  (Topic 46).
- **KV cache** — mandatory (Topic 12); its memory grows with context and concurrency.
- **Context length** — attention cost grows with sequence length; long contexts slow generation and
  consume cache memory.
- **Speculative decoding** — a small draft model proposes tokens, the big model verifies several at
  once. Real speedups when the draft is often right (Phase 13).
- **Flash attention** and similar kernels — faster and more memory-efficient attention.

## 45.5 Running it properly

Operational realities: model load time (seconds to minutes — keep it loaded), memory headroom for the
KV cache, concurrency limits (an unbounded queue OOMs), and health checks that verify the model
actually generates rather than that the process is alive.

And a correctness trap worth knowing: the **chat template** (Topic 14) must match the model. Tools
usually handle it, but a mismatch produces quietly worse output that looks like a bad model.

---

# Part 2 — Questions to implement

Install Ollama and llama.cpp; vLLM if you have an NVIDIA GPU (otherwise read its docs and note the
gap). Build `benchmarks/` here.

### Q1. Ollama in five minutes
**Build:** install, pull a small model, generate via CLI and via the HTTP API.
**Check:** the API is OpenAI-compatible enough that your Phase 4 client works with a base-URL change.
**Explain:** how much of your existing code had to change?

### Q2. Baseline measurements
**Build:** measure, for one model: load time, TTFT, tokens/second (single stream), and memory use.
**Check:** repeat 5 times and report the spread.
**Explain:** these are your reference numbers. Is the speed usable interactively?

### Q3. Quantization sweep — speed and memory
**Build:** the same model at fp16, Q8, Q4, and Q2 (or the nearest available).
**Check:** tabulate memory, tokens/second, and TTFT.
**Explain:** report the curve. Where's the best memory-per-quality point on paper?

### Q4. Quantization sweep — quality
**Build:** run your Phase 9 evaluation set at each quantization level.
**Check:** report the score per level.
**Explain:** where did quality actually break? Did it match the speed/memory sweet spot?

### Q5. Task sensitivity to quantization
**Build:** evaluate Q4 vs fp16 separately on classification, extraction, summarization, and multi-step
reasoning.
**Explain:** which task degraded most? What does that mean for where you can use aggressive
quantization?

### Q6. Big-and-quantized vs small-and-precise
**Build:** at a fixed memory budget, compare a larger model at 4-bit against a smaller one at 16-bit.
**Check:** evaluate both on quality and speed.
**Explain:** which won? Does the §45.2 heuristic hold on your task?

### Q7. Memory bandwidth as the limit
**Build:** measure tokens/second for models of several sizes on the same hardware.
**Check:** plot tokens/second against model size in memory.
**Explain:** is the relationship roughly inverse? Estimate your effective memory bandwidth from the
data.

### Q8. Prefill vs decode
**Build:** measure time to process a 2000-token prompt versus time to generate 2000 tokens.
**Check:** report both, and per-token rates.
**Explain:** report the ratio. Explain it in terms of §45.3.

### Q9. CPU vs GPU, and partial offload
**Build:** run the same model fully on CPU, fully on GPU (if available), and with partial offloading.
**Check:** tokens/second for each.
**Explain:** report the numbers. What happened at the point where the model no longer fit on the GPU?

### Q10. Batching
**Build:** with llama.cpp's server or vLLM, measure total throughput at concurrency 1, 4, 16, 32.
**Check:** report tokens/second aggregate and per-request latency.
**Explain:** how much did aggregate throughput improve? What happened to individual latency, and why
is that trade acceptable in a server?

### Q11. Context length cost
**Build:** measure generation speed and memory at context lengths 512, 4k, 16k, 32k.
**Check:** report both curves.
**Explain:** where does the KV cache become the binding constraint?

### Q12. Chat template mismatch
**Build:** run the same prompt with the correct chat template and with a deliberately wrong one.
**Check:** compare outputs on 10 prompts.
**Explain:** describe the degradation. How would you have misdiagnosed this in the wild?

### Q13. Local vs API, end to end
**Build:** run one real feature from your earlier phases entirely locally.
**Check:** compare quality, latency, and cost per 1000 requests against the API version.
**Explain:** would you ship the local version? For which subset of traffic?

---

# Done when you can answer

1. What is each of Ollama, llama.cpp, vLLM and Transformers for?
2. What's the practical sweet spot for quantization, and what degrades first?
3. Why does generation speed track memory bandwidth?
4. Why is prefill compute-bound while decode is memory-bound?
5. Why does batching improve throughput so much?
6. Bigger-quantized or smaller-precise, at fixed memory?
7. What does a chat template mismatch look like?

Write answers in `notes.md`.
