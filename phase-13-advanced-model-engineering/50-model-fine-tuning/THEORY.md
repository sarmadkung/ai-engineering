# Topic 50 — Model Fine-Tuning

**Why this topic:** Topic 14 taught the mechanics. This topic is the craft: building a dataset good
enough to be worth training on, and knowing whether the result is better. **Data quality dominates
everything else here** — more than model choice, more than hyperparameters.

---

# Part 1 — Theory

## 50.1 Decide before you train

Fine-tuning is justified when: you need a consistent format or style prompting can't hold; you have a
narrow domain with its own conventions; you want a small model to do one thing as well as a big model
(cost and latency); or the behaviour is easier to demonstrate than describe.

It is **not** justified when the problem is missing knowledge (use RAG), when a prompt would do
(cheaper and revisable in seconds), when requirements change weekly, or when you have fewer than a few
hundred consistent examples.

The discipline: **establish a prompted baseline first, with a measured score** (Phase 9). A fine-tune
that doesn't beat a good prompt is a maintenance burden you volunteered for.

## 50.2 Dataset creation

Sources, in descending order of value:

- **Production data** — real inputs with verified good outputs. The best source, and it requires
  logging from day one (Topic 20).
- **Human-written** — expensive, high quality, right for establishing style.
- **Distillation** — a stronger model generates outputs that a smaller model learns to imitate. Cheap
  and effective, and genuinely the standard approach for small task models. Check the provider's terms
  on training from outputs.
- **Synthetic augmentation** — paraphrase inputs, generate variations. Useful for coverage; risks
  amplifying the generator's quirks.
- **Existing datasets** — check licence and relevance.

How many examples: 50–100 for pure format/style, 500–1,000 for a task, 1,000–10,000 for robustness
across variation. **Quality beats quantity decisively** — 200 consistent, correct examples will beat
2,000 noisy ones, and this is not a close call.

## 50.3 Data quality

What matters, in order:

1. **Correctness** — every output must be one you'd be happy to ship. The model learns your errors
   faithfully.
2. **Consistency** — same format, same style, same conventions. **Inconsistency is the single most
   damaging defect**, because it teaches the model to be inconsistent.
3. **Coverage** — the real distribution of inputs, including the boring ones, edge cases, and cases
   where the right answer is a refusal or "insufficient information".
4. **Diversity** — varied phrasings, lengths and difficulty, so the model doesn't latch onto a
   surface pattern.
5. **No leakage** — held-out examples must not appear in training, including near-duplicates
   (Phase 3's lesson).

Process: write an explicit specification of what a good output looks like, review a sample by hand
(always — you will find surprises), deduplicate, and split by source/document rather than randomly.

**Read 50 of your own examples before training.** Every experienced practitioner has a story about
the malformed output that was in 10% of their data.

## 50.4 Formatting

Use the model's chat template (Topic 14) exactly as it will be used at inference — a mismatch between
training and serving format degrades quality in ways that look mysterious.

Mask the prompt, computing loss only on the response (Topic 14). And include the *system prompt you
will actually use*, so the model learns under the conditions it will face.

## 50.5 SFT, practically

Hyperparameters that matter, roughly in order: learning rate (1e-5 to 2e-4 for LoRA, lower for full
fine-tuning), epochs (1–3 — more usually memorizes), LoRA rank and target modules (Topic 14), and
batch size (larger is more stable; use accumulation).

What to watch: training loss should fall smoothly; **validation loss is the one that matters**, and
when it turns upward you are memorizing (Phase 2's lesson, unchanged). Always evaluate on a real task
metric too, because loss and usefulness diverge.

Known failure modes: catastrophic forgetting (general ability decays — mitigate with fewer epochs,
lower learning rate, mixed-in general data), overfitting to phrasing, learning artifacts (a stray
prefix present in most of your outputs), and degraded instruction-following outside the trained task.

## 50.6 Evaluation

The part that makes it engineering rather than hope. You need, all measured on the same held-out set:

- The **prompted baseline** on the original model.
- The fine-tuned model on the **same task**.
- The fine-tuned model on **general capability**, to detect forgetting.
- **Cost and latency** for both, including the frontier-model baseline you're trying to replace.

Then judge honestly: did it beat the prompt? Did it retain general ability? Is the total cost
(training, serving, maintenance, re-training when data drifts) less than the alternative? Fine-tunes
have an ongoing cost that prompts don't — each new base model version is a decision to re-train or
stay behind.

## 50.7 Iteration

Improve the data, not the hyperparameters. The loop:

```
evaluate -> find failure categories -> add/fix examples for them -> retrain -> re-evaluate
```

Most real gains come from fixing data: removing inconsistent examples, adding coverage for a failing
category, correcting wrong outputs. Hyperparameter tuning is a small effect by comparison, and the
time spent there is usually better spent reading data.

Version datasets like code, and record which dataset version produced which model — otherwise you
cannot reproduce or explain a result.

---

# Part 2 — Questions to implement

Build `dataset/` and `train/` here. Use a small model you can train (`Qwen2.5-0.5B`, `TinyLlama`,
`gpt2`) with `peft` + `trl`. Choose one narrow, evaluable task.

### Q1. Justify it
**Build:** a written case: the task, why fine-tuning rather than prompting or RAG, and the success
criterion with a number.
**Explain:** what result would make you abandon the fine-tune?

### Q2. The prompted baseline
**Build:** a well-engineered prompt (Topic 17) on both a small model and a frontier model. Evaluate both
on a 40-case held-out set.
**Check:** report scores, latency, cost.
**Explain:** these are the numbers to beat. Which will be hardest?

### Q3. Build the dataset
**Build:** 300+ examples. Use at least two sources (e.g. distillation from a strong model plus
hand-written edge cases). Write a specification of a good output first.
**Check:** a validator confirming every example matches the spec.
**Explain:** how many examples did your validator reject, and why?

### Q4. Read your data
**Build:** read 50 examples by hand and record every defect.
**Check:** categorize the defects.
**Explain:** report what you found. What fraction was flawed, and what would it have taught the model?

### Q5. Consistency audit
**Build:** measure format consistency automatically (length distribution, structure, opening phrases).
**Check:** find outliers.
**Explain:** show two examples that contradict each other in style. Fix them and say what you changed.

### Q6. Coverage and refusals
**Build:** add cases for edge inputs and for inputs where the correct output is a refusal or "cannot
determine".
**Check:** report your distribution across categories.
**Explain:** which category was missing entirely before you looked?

### Q7. Deduplicate and split
**Build:** near-duplicate detection, then split by source into train/validation/test.
**Check:** no near-duplicate spans the splits.
**Explain:** how many duplicates did you find? What would they have done to your test score?

### Q8. Train
**Build:** LoRA SFT with prompt masking and the correct chat template.
**Check:** training and validation loss curves, plus a task metric per checkpoint.
**Explain:** where did validation loss turn? Which checkpoint did you keep, and why not the last?

### Q9. Evaluate properly
**Build:** the fine-tuned model on your held-out set, against both baselines from Q2.
**Check:** one table — task score, general-capability score, latency, cost.
**Explain:** did you beat the prompted small model? Did you approach the frontier model? At what cost?

### Q10. Catastrophic forgetting
**Build:** a general-capability check (10 unrelated questions) before and after fine-tuning; then again
after deliberately over-training (10 epochs).
**Check:** report all three.
**Explain:** how much general ability did you lose, and was it acceptable?

### Q11. Data quantity ablation
**Build:** train on 50, 150, and all of your examples.
**Check:** report task score per size.
**Explain:** where did returns flatten? What does that say about collecting more data?

### Q12. Data quality ablation
**Build:** train on your cleaned dataset, and on the same size with 20% deliberately inconsistent
examples.
**Check:** report both scores.
**Explain:** report the gap. Is it bigger than the gap from tripling data quantity in Q11? What follows
for where you spend effort?

### Q13. Fix data, not hyperparameters
**Build:** classify your model's failures, add or correct examples targeting the biggest category, and
retrain.
**Check:** re-evaluate; compare with a hyperparameter-tuning attempt of similar effort.
**Explain:** report both improvements. Which was the better use of your time?

### Q14. The lifetime cost
**Build:** total cost of ownership: dataset creation hours, training cost, serving cost, and the cost of
re-training when the base model updates or data drifts.
**Explain:** compare with prompting a frontier model at your expected volume. Which would you ship,
and what would change the answer?

---

# Done when you can answer

1. When is fine-tuning the right tool, and when isn't it?
2. Why must you establish a prompted baseline first?
3. What are the five data-quality properties, and which defect is most damaging?
4. Why is quality decisively more important than quantity?
5. Why must training format match serving format exactly?
6. How do you detect catastrophic forgetting?
7. Why is improving data better than tuning hyperparameters?

Write answers in `notes.md`.
