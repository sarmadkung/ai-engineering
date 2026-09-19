# Phase 13 — Advanced Model Engineering

**Goal:** customize and optimize models properly — building a dataset worth training on, making models
smaller and faster, and understanding what determines whether a training job fits at all.

---

## What I will do

| # | Topic | What I build | Status |
|---|---|---|---|
| 50 | [Model Fine-Tuning](50-model-fine-tuning/) | `dataset/`, `train/` — a fine-tune justified by measurement | ready |
| 51 | [Model Optimization](51-model-optimization/) | `optimize/` — quantization, distillation, pruning, speculation | ready |
| 52 | [Training Systems](52-training-systems/) | `training/` — memory calculators, sharded training, fault tolerance | ready |

## Which things to learn

**50. Model Fine-Tuning** — deciding before you train, with a **prompted baseline** to beat; dataset
sources (production data best, distillation cheapest); the five data-quality properties, with
inconsistency as the most damaging; why **quality beats quantity decisively**; format matching between
training and serving; catastrophic forgetting; evaluating on task *and* general ability; and improving the
data rather than the hyperparameters.

**51. Model Optimization** — PTQ methods (GPTQ, AWQ) and why outliers matter; QAT; distillation by
response versus by logits, and why the teacher's distribution carries more information; what distillation
buys and what it doesn't; structured versus unstructured pruning; **speculative decoding** — lossless,
because verification is parallel while generation is serial; and choosing the technique from the symptom.

**52. Training Systems** — the ~16 bytes per parameter that make training memory-bound, with optimizer
state usually larger than the model; data parallelism; **ZeRO/FSDP sharding**; tensor parallelism inside a
node; pipeline parallelism and the bubble; mixed precision (and why bf16 over fp16); gradient
checkpointing; data loading as an embarrassing bottleneck; MFU; and fault-tolerant runs.

## Prerequisites

Phase 3 (Topics 13–14), Phase 9. A small trainable model, `peft`/`trl`/`accelerate`/`deepspeed`, and
either a GPU or a willingness to work the arithmetic where you can't run something.

**Next:** Phase 14 — reasoning, memory and learning systems.
