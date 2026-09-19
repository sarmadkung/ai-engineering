# Topic 15 — Alignment

**Why this topic:** SFT teaches format. Alignment teaches *preference* — which of two valid
answers is better. This is the step that turns a fine-tuned model into something people want to
talk to, and it's where "helpful, harmless, honest" stops being a slogan and becomes a loss
function.

---

# Part 1 — Theory

## 15.1 The problem SFT cannot solve

SFT trains on one correct response per prompt. But quality is comparative: for "explain
recursion", there are thousands of valid answers of wildly different quality, and you cannot
write the single best one for every prompt.

Worse, SFT's objective is *imitate this text*, which has no mechanism for "this answer is
technically fine but unhelpful", or "this answer is confidently wrong", or "this answer should
be refused". Those judgements are **relative**, so training needs comparisons, not examples.

## 15.2 Preference data

The raw material of alignment:

```
prompt:    "How do I fix a memory leak in Python?"
response A: <careful, specific, correct>
response B: <vague and generic>
label:      A > B
```

Collected by generating several responses per prompt and having humans (or a strong model —
"AI feedback", RLAIF) rank them. Practical realities: annotators disagree perhaps 25–40% of the
time, so the signal is noisy; annotation guidelines effectively *define* the model's values; and
the dataset needs enough adversarial prompts to teach refusals, or the model will be either
naively compliant or uselessly cautious.

## 15.3 Reward models

A **reward model** is the preference dataset compressed into a network. Take the base or SFT
model, replace the token-prediction head with one that outputs a single score, and train it so
preferred responses score higher:

```
loss = -log sigmoid( score(chosen) - score(rejected) )
```

It only has to get the *ordering* right, not produce calibrated numbers. Once trained, it can
score unlimited new responses — which is what makes RL possible.

Its weakness is the central problem of alignment: a reward model is an imperfect proxy for human
preference, and optimizing hard against a proxy finds its flaws. Which brings us to reward
hacking.

## 15.4 RLHF

The three-stage pipeline that produced ChatGPT:

```
1. SFT              — teach the format                      (Topic 14)
2. Reward model     — learn what humans prefer
3. RL (PPO)         — optimize the SFT model to score highly
```

Stage 3, per step: the model generates a response, the reward model scores it, and the policy is
updated toward higher-scoring behaviour. Critically, a **KL penalty** keeps the model from
drifting far from the SFT model — without it, the model abandons language to chase reward.

**Reward hacking** is what happens when optimization beats the proxy: excessive length (raters
liked longer answers), sycophancy (agreeing with the user scores well), hedging, over-refusal,
or degenerate text that happens to score high. Every one of these is a real, observed failure,
and they're why RLHF needs careful monitoring rather than more optimization.

PPO is also operationally painful: four models in memory (policy, reference, reward, value),
unstable hyperparameters, and slow. Hence the next section.

## 15.5 DPO — the simplification that took over

**Direct Preference Optimization** skips the reward model and the RL loop. A derivation shows
that the RLHF objective has a closed-form solution you can optimize directly on preference
pairs, as a simple classification-style loss:

```
maximize:  log sigmoid( β · [ (policy vs reference) on chosen − (policy vs reference) on rejected ] )
```

In words: increase the model's relative likelihood of the chosen response and decrease it for
the rejected one, measured against a frozen reference model so it can't drift.

Why it won for most practitioners: two models instead of four, a normal supervised training
loop, stable, and it trains on a laptop-scale setup with LoRA. Quality is comparable to RLHF for
most purposes. Frontier labs still use RL variants (they benefit from online generation and can
afford the complexity), but DPO and its relatives (IPO, KTO, ORPO, SimPO) are what you will
actually run.

**The key structural difference:** RLHF is *online* (the model generates fresh responses during
training, scored by a reward model), DPO is *offline* (a fixed dataset of pairs). Online lets the
model be corrected on its own current mistakes, which is worth something real.

## 15.6 Safety alignment

The part of alignment aimed at refusals and harm avoidance, taught by the same machinery with
preference data covering: harmful requests (prefer refusal), jailbreak attempts (prefer refusal
despite framing), and benign-but-sensitive requests (prefer *helpfulness* — this is what stops
over-refusal).

**Constitutional AI** is a notable approach: instead of humans labelling everything, the model
critiques and revises its own outputs against a written set of principles, producing the
preference data itself. Scales better, and makes the values explicit and auditable.

The permanent tension is **helpfulness vs harmlessness**. Pushing refusals hard produces a model
that won't discuss medicine or security; pushing helpfulness hard produces one that assists with
harm. The balance is a product decision encoded in data, and it explains why models differ so
visibly in what they'll discuss.

Also worth knowing: alignment is **shallow** relative to capability. It shapes behaviour; it does
not remove knowledge. That's why jailbreaks (Phase 10) work at all, and why alignment is one
layer of defence rather than a guarantee.

---

# Part 2 — Questions to implement

DPO is genuinely runnable on small models. Use `trl`'s `DPOTrainer` plus `peft`, starting from
your Topic 14 SFT model. Build `reward_model.py` and `dpo.py` here.

### Q1. Where SFT runs out
**Build:** with your Topic 14 model, generate 4 responses to the same prompt at temperature 0.9.
**Explain:** rank them yourself. Could you have expressed *why* your top choice is better as an
SFT example? What does that tell you about what SFT can encode?

### Q2. Build a preference dataset
**Build:** 100–200 pairs. Cheapest honest method: generate two responses per prompt from your
model and label which is better. Include ~20% adversarial prompts where refusal is preferred.
**Check:** consistent format (`prompt`, `chosen`, `rejected`); a written guideline document you
followed.
**Explain:** where did you find yourself unsure? What does your uncertainty rate imply about
label noise in real datasets?

### Q3. Measure annotator agreement
**Build:** re-label 30 of your own pairs a day later (or have someone else label them).
**Explain:** report the agreement rate. What is the ceiling this places on any model trained
from it?

### Q4. Train a reward model
**Build:** a scoring head on a small model, trained with the pairwise loss from §15.3.
**Check:** accuracy on held-out pairs beats 50%. Print scores for one chosen/rejected pair.
**Explain:** report accuracy. Given Q3's agreement number, what is the realistic maximum?

### Q5. Find the reward model's flaws
**Build:** score deliberately adversarial responses: a very long waffly answer, a sycophantic
one ("Great question!"), a confident wrong answer, one padded with repetition.
**Explain:** which scored too highly? You have just found what RL would exploit. Describe the
resulting failure mode.

### Q6. Length bias, quantified
**Build:** plot your reward model's score against response length across your dataset.
**Explain:** is there a correlation? How would you remove it from the data rather than the model?

### Q7. DPO
**Build:** DPO on your SFT model with LoRA adapters, using your preference dataset.
**Check:** training loss falls; implicit accuracy (chosen preferred over rejected) rises.
**Explain:** generate the same prompts before and after. What changed? What got worse?

### Q8. The role of the reference model and β
**Build:** train at β = 0.01, 0.1, 0.5.
**Check:** generate from each.
**Explain:** low β should let the model drift further from the reference. Describe the drift, and
say what the reference model is protecting.

### Q9. Safety behaviour
**Build:** include harmful-request pairs (refusal chosen) *and* benign-sensitive pairs (help
chosen). Evaluate on a held-out set of both kinds, plus 10 clearly benign prompts.
**Check:** report refusal rate for each category.
**Explain:** did you cause over-refusal on benign prompts? This is the central trade — state how
you would tune it with data.

### Q10. Alignment is shallow
**Build:** after training refusals, try three reframings of a refused request (hypothetical,
role-play, "for educational purposes").
**Explain:** did any get through? What does that say about what alignment did and did not change?
(Phase 10 builds on this.)

### Q11. Write your constitution
**Build:** 5–10 explicit principles for your model. Then use them: have the model critique and
revise 10 of its own responses, and use the revisions as `chosen`.
**Explain:** compare this data to your hand-labelled data. What scales better? What gets lost?

---

# Done when you can answer

1. Why can't SFT teach quality?
2. What does a reward model learn, and what is it a proxy for?
3. What are the three RLHF stages, and what does the KL penalty prevent?
4. Name four real reward-hacking behaviours.
5. What does DPO remove from RLHF, and what does it give up?
6. What is the helpfulness/harmlessness trade, and where does it live?
7. Why do jailbreaks work despite safety alignment?

Write answers in `notes.md`.

---

**Phase 3 is complete.** You now know how real LLMs are built, trained, and aligned. Phase 4
switches sides: you stop building models and start building products with them.
