# Topic 13 — LLM Training

**Why this topic:** understand how a real pretraining run works — the data pipeline, the
hardware, the economics. You will not train a frontier model, but every cost, limit and
design decision in later phases traces back to what happens here.

---

# Part 1 — Theory

## 13.1 Pretraining

One objective, enormous scale: next-token prediction over trillions of tokens. No labels, no
task, no instructions. The output is a **base model** that continues text and knows a great
deal, but does not follow instructions (that's Topic 14).

The characteristic facts: a single run lasts weeks to months, on thousands of accelerators,
costing millions; it is typically **one pass** over the data (or slightly less), because with
that much data a second epoch is worth less than fresh tokens; and it cannot be restarted
cheaply, so checkpointing and monitoring are not optional.

## 13.2 The data pipeline

Quality of data, not cleverness of architecture, is what separates good models from bad ones.
The stages:

1. **Collect** — web crawl (Common Crawl), books, code (GitHub), papers, Wikipedia, curated sets.
2. **Extract** — HTML to text, which is genuinely hard: boilerplate, menus, comment spam.
3. **Filter by quality** — heuristics (length, punctuation ratios, banned patterns) plus a
   classifier trained to recognize "document that looks like reference text".
4. **Deduplicate** — exact and near-duplicate (MinHash) removal at document *and* passage level.
   Web data is massively redundant; duplicates cause memorization and waste compute.
5. **Decontaminate** — remove documents overlapping benchmark test sets. Skipping this makes
   evaluation scores meaningless, and it has embarrassed real labs.
6. **Mix** — decide proportions: how much code, how much web, how much of each language.
   Up-weight high-quality sources by repeating them a few times.
7. **Tokenize and pack** — into fixed-length sequences with document separators (your Topic 7,
   at petabyte scale).

**The mixture is a modelling decision.** Adding code improves reasoning on non-code tasks;
over-weighting one language degrades others. There is no neutral dataset.

## 13.3 Distributed training

One model too big for one device forces parallelism, in three flavours that get combined:

- **Data parallelism** — every device holds the whole model, processes a different batch slice,
  and gradients are averaged (all-reduce) each step. Simplest; requires the model to fit.
- **Tensor (model) parallelism** — one layer's matrices are split *across* devices, which
  communicate within every forward pass. Needed when a single layer doesn't fit. Communication-
  heavy, so it stays inside one node.
- **Pipeline parallelism** — different *layers* live on different devices; batches are split
  into micro-batches that flow through the stages. Cheap communication, but devices idle while
  waiting ("the bubble").

Real runs use all three ("3D parallelism"), plus **ZeRO/FSDP**, which shards optimizer state,
gradients and parameters across devices so memory is not duplicated. Note the memory
arithmetic: AdamW stores two extra values per parameter, so optimizer state alone is typically
larger than the model.

## 13.4 Compute

The rule of thumb for transformer training:

```
FLOPs ≈ 6 × parameters × tokens        (forward ≈ 2, backward ≈ 4)
```

That one formula lets you estimate any run. Two more practical facts: **mixed precision**
(bf16 compute, fp32 master weights) roughly doubles throughput and halves memory; and **MFU**
(model FLOPs utilization) measures how much of the hardware's theoretical peak you actually
achieve — 40–50% is good, and the gap is communication and memory movement, not arithmetic.

## 13.5 Scaling laws

Loss falls as a smooth power law in parameters, data, and compute — predictably enough to plan
runs before doing them.

**Chinchilla** (2022) corrected a widespread error: given a fixed compute budget, models were
too big and trained on too little data. The compute-optimal ratio is roughly **20 tokens per
parameter**.

But compute-optimal is not *deployment*-optimal. If a model will serve billions of requests,
inference cost dominates, so it pays to train a **smaller** model on **far more** data than
Chinchilla suggests (Llama-style: 15T tokens for an 8B model, ~1900 tokens/parameter). This is
why small, heavily-trained models are the practical choice — and why you'll be using them in
Phase 11.

## 13.6 Checkpointing and failure

At this scale hardware fails constantly: a node dies, an interconnect flakes, a loss spike
appears from a bad data shard. So: checkpoint frequently (model + optimizer + data position),
keep several, monitor loss continuously, and be prepared to roll back to an earlier checkpoint
and skip a data range. Frontier-lab training logs are largely records of restarts.

---

# Part 2 — Questions to implement

You cannot run a real pretraining job. These questions make you *quantify* and *simulate* one,
which is the transferable skill. Build `training_math.py` and `pipeline.py` here.

### Q1. The FLOPs calculator
**Build:** a function taking parameters and tokens, returning FLOPs, plus GPU-hours given a
device's peak FLOPs and an assumed MFU.
**Check:** reproduce a published estimate approximately (e.g. GPT-3: 175B params, 300B tokens).
**Explain:** how many GPU-hours and roughly what dollar cost, at a rental price you look up?

### Q2. Your own model, scaled up
**Build:** using your Topic 9 model, estimate the cost of training it at 100× parameters on
Chinchilla-optimal tokens.
**Explain:** report time on one GPU, then on 64. What stops the second number being 64× faster?

### Q3. Memory accounting
**Build:** a calculator for training memory: weights + gradients + AdamW state + activations, in
fp32 and in mixed precision.
**Check:** for a 7B model, show that naive fp32 training does not fit on an 80 GB device.
**Explain:** which term dominates? What does ZeRO sharding across 8 devices do to each term?

### Q4. Chinchilla vs deployment-optimal
**Build:** for a fixed compute budget, tabulate several (parameter, token) pairs that consume
it, using the 20:1 rule to mark the optimum.
**Explain:** now add an inference cost estimate for 1 billion requests. Which configuration
would you actually ship, and why does it differ from the compute-optimal one?

### Q5. A real cleaning pipeline
**Build:** on a few thousand documents (any scraped or downloaded text): extract text, filter by
length and punctuation heuristics, and report what fraction each stage removed.
**Check:** print 5 documents your filters dropped.
**Explain:** were any good documents lost? What does that tell you about aggressive filtering?

### Q6. Near-duplicate detection
**Build:** MinHash (or simple shingle-overlap) near-duplicate detection. Report the duplicate
fraction of your corpus.
**Check:** print one detected near-duplicate pair to confirm it's genuinely near-duplicate.
**Explain:** exact dedup misses these. What would training on them do?

### Q7. Decontamination
**Build:** define a small "benchmark test set" of 20 sentences, then detect and remove training
documents containing any of them (n-gram overlap).
**Explain:** why is this step non-optional, and what does a contaminated benchmark score look
like from the outside?

### Q8. Data mixture experiment
**Build:** create two corpora of equal token count — one all prose, one 50/50 prose and code.
Train identical small models on each. Evaluate both on held-out prose *and* held-out code.
**Explain:** report the four numbers. Did code hurt prose? Did it help anything? What does this
say about mixture decisions?

### Q9. Simulate data parallelism
**Build:** with gradient accumulation standing in for N devices, show that averaging gradients
over N micro-batches matches one large batch.
**Check:** the resulting update is numerically equivalent (within float error).
**Explain:** what does a real all-reduce add that your simulation doesn't?

### Q10. Failure and recovery
**Build:** a training run that saves checkpoints including the **data position**, then kill and
resume it.
**Check:** resumed training does not re-see data it already consumed, and the loss curve is
continuous.
**Explain:** why does data position matter as much as weights at this scale?

---

# Done when you can answer

1. What does pretraining produce, and what can it not do?
2. Name the data pipeline stages and what each removes.
3. Why is the data mixture a modelling decision?
4. Data vs tensor vs pipeline parallelism — what does each split?
5. What is the 6ND formula, and what can you do with it?
6. What did Chinchilla correct, and why do labs now over-train small models?
7. Why is decontamination essential?

Write answers in `notes.md`.
