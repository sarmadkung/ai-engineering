# Topic 14 — Fine-Tuning

**Why this topic:** a base model continues text; a useful model follows instructions. This is
the step that makes that change, and the first one you can actually run yourself on a laptop
or a cheap rented GPU.

---

# Part 1 — Theory

## 14.1 Transfer learning

Pretraining is expensive and general; fine-tuning is cheap and specific. Start from
pretrained weights and continue training on a small, targeted dataset. The model keeps its
language ability and adapts its behaviour.

Scale comparison worth internalizing: pretraining is trillions of tokens; fine-tuning is
thousands to millions of examples. The lower learning rate (typically 10–100× lower than
pretraining) exists for the same reason — you are nudging, not teaching from scratch.

## 14.2 Instruction tuning: what actually changes

The base model's problem isn't knowledge, it's **format**. It has seen the answer to your
question somewhere; it just doesn't know that a question should be followed by an answer.

Instruction tuning trains on pairs shaped like the interaction you want:

```
### Instruction:
Summarize this paragraph.
### Input:
<text>
### Response:
<summary>
```

After a few thousand such examples, the model reliably produces answers instead of more
questions. Note what this implies: instruction tuning mostly teaches **behaviour and format**,
not new facts. Trying to add knowledge this way largely fails — that's what RAG (Phase 5) is
for, and confusing the two is one of the most common mistakes in applied AI.

## 14.3 Supervised fine-tuning (SFT), mechanically

Same loss as pretraining — next-token prediction — with one crucial difference: **mask the
prompt**. Compute loss only on the response tokens.

```
tokens:  [### Instruction: ... ### Response:]  [the answer tokens]
loss on:  ignored (masked)                      computed
```

Without masking, the model spends capacity learning to generate instructions, which you never
want it to do.

Also here: **chat templates**. Multi-turn conversations are serialized with special tokens
marking each role (`<|user|>`, `<|assistant|>`). The template is part of the model's contract —
using the wrong one at inference visibly degrades quality, which is a frequent cause of "this
open model is bad" complaints.

**Catastrophic forgetting** is the main risk: over-train on a narrow set and general ability
decays. Mitigations: few epochs (1–3), low learning rate, and mixing in some general data.

## 14.4 LoRA — the key practical technique

Full fine-tuning updates every weight, which needs memory for weights + gradients + optimizer
state — typically 12–16 bytes per parameter. A 7B model is out of reach on consumer hardware.

**LoRA (Low-Rank Adaptation)** freezes the pretrained weights and trains a small additive
update, factored into two thin matrices:

```
frozen:    W        (d × d)
trained:   A (d × r), B (r × d)       output = W x + B A x,   r is small (8, 16, 64)
```

If `d = 4096` and `r = 16`, that's ~0.4% as many trained parameters. The intuition: the
*adaptation* a task needs is far simpler than the knowledge already in W, so a low-rank update
suffices.

Consequences that make it dominant in practice:
- Trainable parameters drop 100–1000×, and optimizer state drops with them.
- Adapters are tiny files (megabytes), so you can keep dozens of behaviours for one base model
  and swap them per request.
- Adapters can be **merged** into the base weights afterwards for zero inference overhead.
- Two knobs: `r` (capacity) and which modules to target (attention projections at minimum;
  adding the FFN helps more and costs more).

## 14.5 QLoRA

LoRA still needs the frozen base model in memory. **QLoRA** loads it **quantized to 4 bits**
and trains LoRA adapters in higher precision on top.

```
7B in fp16  ≈ 14 GB      7B in 4-bit ≈ 4 GB
```

This is what puts fine-tuning a 7B–13B model on a single consumer GPU. Cost: slightly slower
per step (dequantization) and a small quality loss versus LoRA — usually an easy trade.

## 14.6 PEFT and the library landscape

**PEFT** (parameter-efficient fine-tuning) is the umbrella term: LoRA, QLoRA, prefix tuning,
prompt tuning, adapters, DoRA. LoRA family dominates because it's simple and merges cleanly.

Practically you'll use Hugging Face `peft` + `transformers` + `trl`, or one of the wrappers
(Axolotl, Unsloth). Know what they're doing underneath — that's what this topic is for.

## 14.7 When *not* to fine-tune

The most valuable judgement in this topic. Fine-tuning is the wrong tool when:

- You need **facts** the model lacks → use RAG (Phase 5).
- You need **behaviour** you could specify in a prompt → try prompting first (Phase 4); it is
  free, instant, and revisable.
- Your data is **small or inconsistent** → you'll teach noise.
- Your requirements **change weekly** → prompts change in seconds, fine-tunes take hours.

Fine-tune for: consistent format/style, a narrow domain's tone and vocabulary, latency and cost
(a small fine-tuned model replacing a large prompted one), and behaviours too complex to
specify in words but easy to demonstrate.

---

# Part 2 — Questions to implement

You need a real base model. Use a small one (`gpt2`, `Qwen2.5-0.5B`, `TinyLlama`) so it runs on
your machine. Install `transformers`, `peft`, `trl`, `datasets`, `bitsandbytes`. Build
`sft.py` and `lora.py` here.

### Q1. Prove the base model's problem
**Build:** load a base (non-instruct) model. Prompt it with a direct question, and with a
few-shot prompt showing two Q→A examples.
**Explain:** compare the outputs. Describe the base model's failure precisely, and say what
the few-shot prompt supplied that the question alone did not.

### Q2. Build an SFT dataset
**Build:** 100–500 instruction/response examples in a consistent format — either curated from
an existing dataset or written for a narrow task you care about.
**Check:** every example has the same structure; you have a train/validation split.
**Explain:** what task did you choose, and how will you judge success *before* training?

### Q3. Prompt masking
**Build:** tokenize your examples so loss is computed only on response tokens (label `-100` on
prompt positions).
**Check:** print one example's tokens beside its labels and verify by eye where masking ends.
**Explain:** train briefly *without* masking and describe what the model starts doing.

### Q4. Full fine-tune a small model
**Build:** SFT the smallest model you have, fully. Track train and validation loss.
**Check:** it now answers in your format.
**Explain:** report peak memory and time. Extrapolate to 7B — what would it need?

### Q5. Catastrophic forgetting
**Build:** before and after fine-tuning, ask 5 general-knowledge questions unrelated to your
task.
**Explain:** did general ability degrade? Then over-train deliberately (10 epochs) and re-check.
What did that cost you?

### Q6. LoRA
**Build:** the same fine-tune with LoRA (`r=16`, targeting attention projections).
**Check:** print trainable vs total parameters and the percentage. Compare loss curve, memory
and time against Q4.
**Explain:** did quality hold? What did you save?

### Q7. Rank and target sweep
**Build:** LoRA at `r` = 4, 16, 64, and with/without FFN modules targeted.
**Explain:** tabulate trainable parameters and validation loss. Where do the returns stop? Which
target choice mattered more?

### Q8. Adapter swapping and merging
**Build:** train two adapters for two different behaviours on one base model. Load each in turn.
Then merge one into the base weights and save a standalone model.
**Check:** merged model output matches adapter-loaded output. Adapter files are megabytes; the
merged model is gigabytes.
**Explain:** when would you ship an adapter, and when a merged model?

### Q9. QLoRA
**Build:** load the base in 4-bit and train adapters on top.
**Check:** report memory versus Q6, and step time.
**Explain:** what quality difference did you observe, and was the memory saving worth it?

### Q10. Chat template
**Build:** format a multi-turn conversation using the model's own chat template, then again with
a wrong/ad-hoc format. Generate from both.
**Explain:** describe the difference. Why is the template part of the model's contract?

### Q11. The judgement call
**Build:** pick one real task. Solve it three ways: (a) a careful prompt on a larger model, (b)
LoRA on a small model, (c) prompt plus retrieved context (stub the retrieval).
**Explain:** compare quality, latency, cost per 1000 calls, and how long each took you to build.
Which would you ship, and under what circumstances would you switch?

---

# Done when you can answer

1. What does instruction tuning actually change about a model?
2. Why mask the prompt during SFT?
3. What does LoRA train instead of the full weights, and why is low rank enough?
4. What does QLoRA add, and what does it cost?
5. What is catastrophic forgetting, and how do you avoid it?
6. When should you use RAG instead of fine-tuning?
7. Why does the chat template matter at inference?

Write answers in `notes.md`.
