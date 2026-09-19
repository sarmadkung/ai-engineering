# Topic 58 — AGI Research Topics

**Why this topic:** the open questions the field is actually arguing about. Knowing them lets you read
research critically, judge claims, and anticipate what will change about your engineering. Everything
here is contested — treat confident answers, including your own, with suspicion.

---

# Part 1 — Theory

## 58.1 Scaling

The central empirical fact of the last decade: performance improves predictably with more parameters,
data and compute — smoothly, over many orders of magnitude (Topic 13).

The live questions:

- **Does it continue?** Loss keeps falling; whether that keeps producing *useful* capability gains is
  disputed.
- **Have we run out of data?** High-quality text is finite. Responses: synthetic data (with error
  amplification risk — Topic 55), multimodal data, and more passes over curated data.
- **Has the axis shifted?** Recent gains have come substantially from post-training and **test-time
  compute** (Topic 53) rather than pretraining scale — a real change in where the field spends money.
- **Economics** — the next order of magnitude costs an order of magnitude more, and inference cost
  matters more than training cost when a model serves billions of requests (Topic 13's
  deployment-optimal argument).

Why this matters to you: if capability keeps improving quickly, invest in scaffolding that's easy to
throw away. If it plateaus, invest in domain-specific engineering. Most teams should build so that
either outcome is survivable — which mostly means keeping prompts, evaluation and business logic
separate from model-specific hacks.

## 58.2 Emergent capabilities

Capabilities that appear at scale without being designed: in-context learning, chain-of-thought
benefit, instruction following, tool use.

The important caveat, and it's a genuine correction to earlier excitement: apparent **sharp** emergence
is often an artifact of discontinuous metrics. Measure with a smooth metric (per-token probability
rather than exact match) and many "phase transitions" become gradual curves. Some sharpness survives
this analysis; much doesn't.

What's not in dispute: capabilities *do* appear that nobody predicted, and predicting which capability
appears at which scale remains unsolved. That unpredictability is itself a safety-relevant fact
(Topic 59).

## 58.3 Reasoning

The frontier question: is there a ceiling to what next-token prediction plus test-time compute can
reason about?

What's established: reinforcement learning against verifiable outcomes produces large gains on maths
and code; test-time compute trades predictably for accuracy; and this works best where verification is
cheap (Topic 53).

What's open: whether the same approach extends to domains without cheap verification (law, medicine,
strategy, research), whether reasoning transfers across domains or stays narrow, and whether models
are performing search or sophisticated pattern completion — with the honest answer being that the
distinction may not be as clean as the question assumes.

## 58.4 Continual learning

Covered in Topic 55; the research-level framing: all of catastrophic forgetting, the
stability/plasticity trade-off, credit assignment from sparse real-world feedback, and safety
(a model that changes continuously cannot be certified) remain unsolved.

It's the most obvious structural difference between current systems and anything that learns over a
lifetime. Approaches under investigation: parameter-efficient continual updates, replay, modular
adapters, and better memory as a substitute for weight updates.

## 58.5 Embodied intelligence

The claim: meaning requires grounding in sensorimotor interaction, so text-only systems have a ceiling
— especially for causality, since intervening in the world teaches what observing it cannot.

Progress: vision-language-action models for robotics, simulation-to-real transfer, and LLMs as high-level
planners for robotic systems. Difficulties: data is scarce and expensive compared with text, the real
world is slow and unforgiving, and long-horizon manipulation remains hard.

The counter-argument is that human language already encodes an enormous amount of grounded experience,
so text may be a compressed channel for it. Nobody has settled this.

## 58.6 Multimodal intelligence

Beyond a text model with an image encoder bolted on: unified models handling text, images, audio and
video natively (Phase 16).

Why it might matter for generality: more of the world's information is not text, cross-modal grounding
may improve concepts, and interaction with the physical world requires perception. Open questions:
whether unified beats specialist, how to align representations across modalities, and whether
multimodality produces capabilities beyond the sum of the parts.

## 58.7 Other live areas

- **Mechanistic interpretability** — reverse-engineering what a network computes. Circuits,
  superposition, sparse autoencoders for feature extraction. Relevant to safety, alignment, and
  debugging.
- **Sample efficiency** — humans learn from far less. The largest gap on the skill-acquisition framing
  (Topic 56).
- **Self-verification** — models reliably checking their own work. Would unlock autonomy everywhere
  that lacks cheap verification (Topic 36).
- **Long-horizon agency** — coherent goal pursuit over days or months.
- **Efficiency** — same capability, less compute: distillation, sparsity, new architectures
  (state-space models as an alternative to attention).

## 58.8 How to follow this field

A practical method, because the volume is unmanageable and most of it doesn't matter:

- Read the **primary papers** for things you'll actually use; skim the rest.
- Distrust single-benchmark claims (contamination, cherry-picking — Topic 37).
- Prefer results that **replicate** and are evaluated on held-out data.
- Watch for the pattern: a capability declared impossible, then achieved by scale; and the reverse, a
  capability claimed from a demo that doesn't survive evaluation.
- **Test claims on your own tasks.** Your evaluation set is a better instrument than any paper's
  numbers for deciding what to adopt.
- Track the people and labs whose track records are good, rather than the loudest.

---

# Part 2 — Questions to investigate

Reading, replication and analysis. Keep notes with sources and dates in `notes.md`.

### Q1. Read the scaling literature
**Build:** read the Kaplan scaling-laws paper and the Chinchilla paper.
**Explain:** what exactly did Chinchilla correct, and why did the error matter commercially? Then state
what current practice does differently again, and why (Topic 13).

### Q2. Test the emergence-is-an-artifact claim
**Build:** take a task where a small model scores 0 by exact match. Score the same outputs with a smooth
metric (per-token probability of the correct answer, or partial credit).
**Check:** compare the two curves across the model sizes available to you.
**Explain:** did smooth measurement change the shape? What does that suggest about emergence claims?

### Q3. Replicate a paper's result
**Build:** pick a small, recent, practical result (a prompting technique, a retrieval method) and
reproduce it on your own data.
**Check:** compare your numbers with the paper's.
**Explain:** did it replicate? If not, what differed — data, model, or measurement?

### Q4. Contamination check
**Build:** for a public benchmark, search for its test items in a model's outputs (ask it to complete a
partial test question).
**Check:** does it recall the item?
**Explain:** what does contamination do to reported progress, and how would you detect it as a reviewer?

### Q5. Verification-cheap vs verification-expensive domains
**Build:** the same reasoning approach (high effort plus self-consistency) on maths/code and on a
judgement domain without objective answers.
**Check:** measure the gain in each.
**Explain:** report both. Does §58.3's claim hold in your data?

### Q6. Sample efficiency, measured
**Build:** teach a novel task in-context with 1, 3, 10, 30 examples.
**Check:** report the learning curve.
**Explain:** how many examples would a person need? What does the gap say about the
skill-acquisition framing?

### Q7. Self-verification limits
**Build:** on 30 problems with known answers, have the model (a) answer and (b) verify its own answer.
**Check:** report answer accuracy and verification accuracy separately, including how often it approves
a wrong answer.
**Explain:** report the false-approval rate. What would reliable self-verification unlock?

### Q8. Interpretability, hands on
**Build:** if you have local model access, probe for a simple internal representation — e.g. train a
linear probe on hidden states to predict something the model was never explicitly trained to output.
**Check:** report probe accuracy against a chance baseline.
**Explain:** what does a successful probe show, and what does it not show?

### Q9. A causal intervention test
**Build:** compare the model's accuracy on observational questions versus intervention questions in the
same domain (extend Topic 56, Q7 with a larger set).
**Explain:** report the gap. Is it consistent with the grounding argument in §58.5?

### Q10. Multimodal grounding
**Build:** ask a spatial or physical question in text alone, and then with an image showing the same
situation (Phase 16's models).
**Check:** compare accuracy.
**Explain:** did the image help more than a verbal description of it? What does that suggest about
modality-specific grounding?

### Q11. Efficiency frontier
**Build:** track down the smallest model you can find that passes your Phase 9 evaluation bar, and
compare it with the frontier model's size.
**Explain:** how much has the efficiency frontier moved? What would the same task have required two
years ago?

### Q12. Read across camps
**Build:** read one recent paper each from: scaling, reasoning/RL, interpretability, and
embodiment/world models.
**Explain:** summarize each and state its strongest claim and weakest evidence.

### Q13. Build a claim-testing habit
**Build:** take three claims currently circulating about model capability and test each on your own
evaluation set.
**Check:** report what you found.
**Explain:** which claims survived contact with your data?

### Q14. Your research agenda
**Write:** if you had a year and a small team, what open question from this topic would you work on?
**Explain:** why that one, what experiment you'd run first, and what result would tell you to stop.

---

# Done when you can answer

1. What is established about scaling, and what is contested?
2. Why are sharp emergence claims often artifacts?
3. Where does RL-for-reasoning work, and where is it unproven?
4. Why is continual learning unsolved?
5. What is the grounding argument, and the counter-argument?
6. What would reliable self-verification unlock?
7. How do you evaluate a research claim without taking it on trust?

Write answers in `notes.md`.
