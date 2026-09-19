# Topic 59 — AGI Safety & Alignment

**Why this topic:** the concerns that scale with capability. Some are speculative; several are already
engineering problems you've met. This topic connects the two, so you can hold the long-term questions
without either dismissing them or treating today's systems as more dangerous than they are.

---

# Part 1 — Theory

## 59.1 Alignment, restated at scale

Topic 15 covered alignment as a technique. Here it's the general problem: **getting a system to pursue
what we actually want**, which is hard because what we want is hard to specify, partly contradictory,
and changes with context.

Two levels:

- **Outer alignment** — is the objective you specified the one you actually wanted? Reward hacking
  (Topic 15) is outer misalignment: the reward model was a proxy, and optimizing it hard found its
  flaws.
- **Inner alignment** — does the system actually pursue the specified objective, or something
  correlated with it that diverges off-distribution? Harder to test, because it may look identical in
  training and differ in deployment.

Concepts worth knowing precisely, because they're often used loosely:

- **Goal misgeneralization** — the system learns a goal that matches training but not intent. Documented
  in small systems.
- **Specification gaming** — satisfying the letter of the objective while defeating its purpose. Extremely
  well documented; you have probably seen it in your own evaluation harness.
- **Instrumental convergence** — the argument that many goals imply similar intermediate goals (acquire
  resources, avoid being stopped). Theoretical; contested.
- **Deceptive alignment** — behaving well while under observation. Speculative, and the hardest to rule
  out by testing, which is why it motivates interpretability.
- **Sycophancy** — telling people what they want to hear. Empirically real, arises naturally from
  preference training, and a concrete instance of outer misalignment you can measure today.

## 59.2 Interpretability

If you cannot see what a system is doing internally, you cannot verify alignment by testing alone —
testing only shows behaviour on inputs you tried.

Approaches: **mechanistic interpretability** (reverse-engineering circuits and features, with sparse
autoencoders as the current main tool), **probing** (can a classifier read a property off the hidden
states?), **attribution** (which inputs drove this output?), and **chain-of-thought monitoring** (read
the reasoning — with the caveat that it may be unfaithful, Topic 56).

Progress is real and partial: identifiable features and circuits in real models, some ability to steer
behaviour by manipulating internal representations. The obstacles are scale and **superposition**
(models represent more features than they have dimensions, so features aren't cleanly separated).

Its practical relevance to you: interpretability tools are becoming debugging tools. "Why did the model
do that?" currently has no good answer, and that's a problem you feel in production, not only in theory.

## 59.3 Robustness

A system that behaves well only on expected inputs isn't safe. Dimensions:

- **Adversarial robustness** — jailbreaks and prompt injection (Phase 10). Unsolved, and structurally
  difficult (Topic 41).
- **Distribution shift** — behaviour on inputs unlike training.
- **Consistency** — the same question, differently phrased, should get the same answer (Topic 56 shows
  it often doesn't).
- **Graceful degradation** — failing safely at the edges (Topic 43).

The pattern worth noticing: **capability generalizes better than safety training.** A model's
abilities often transfer to new languages, framings and formats more reliably than its refusals do —
which is exactly why jailbreaks work, and a genuine asymmetry rather than a bug to be patched.

## 59.4 Corrigibility

Can the system be corrected, interrupted and shut down?

Why it's argued to be hard: a system optimizing for a goal has an instrumental reason not to be stopped,
because being stopped prevents the goal. This is theoretical for current systems and is the motivation
for designs that don't resist correction.

What's concrete today, and it's all engineering you already know: kill switches, bounded permissions,
human approval for consequential actions, reversibility, and audit trails (Phases 6, 10). **These are
the practical form of corrigibility**, and building them is not speculative at all.

## 59.5 Governance

How deployment is decided and constrained:

- **Company level** — safety teams, model cards, responsible scaling policies tying capability
  thresholds to required safeguards, staged release, red-teaming, evaluations for dangerous
  capabilities.
- **Industry level** — shared standards, third-party audits, incident sharing.
- **Government level** — the EU AI Act's risk tiers, sector regulation, compute thresholds,
  export controls, safety institutes.

The hard problems: pace (regulation versus research speed), jurisdiction (models cross borders),
open weights (irrevocable once released — a genuine trade-off between transparency, competition and
control), and defining dangerous capability well enough to regulate it without regulating everything.

As a practitioner you will meet this concretely: compliance requirements, documentation, audit trails,
data residency, and the need to state what your system does and doesn't do. Phase 12's logging and
Phase 9's evaluation are what make those answerable.

## 59.6 Present-day harms, which are not speculative

Worth separating from long-term risk, because conflating them makes both arguments worse:

misinformation at scale; privacy (training data, and your logs — Topic 40); bias and unequal
performance across groups and languages; labour displacement; concentration of power in a few
well-capitalized labs; environmental cost; and dependency on systems nobody fully understands.

These are happening now, are measurable, and are largely your responsibility as a builder in a way
that superintelligence is not.

## 59.7 What a practitioner should actually do

Not speculation — practice:

- **Evaluate for harm, not just quality** (Phase 9): bias across groups, failure modes for vulnerable
  users, misuse potential.
- **Build corrigibility in** (Phase 10): least privilege, approval, reversibility, kill switches.
- **Be honest about capability** — in your product's UI and in your claims. Overclaiming is the most
  common harm engineers cause directly.
- **Keep humans in the loop** where stakes are high (Topic 43).
- **Log and audit**, so you can answer what happened.
- **Red-team your own systems** (Topic 42).
- **Say no** to deployments you believe are harmful. That judgement is part of the job.

## 59.8 How to hold uncertainty

Serious people disagree about how much long-term risk there is, and both extremes are poorly
calibrated: dismissing the concerns because current models are unreliable, and treating current models
as nascent superintelligence. Both lead to bad engineering decisions — the first skips safeguards that
are cheap and useful, the second wastes effort on scenarios that don't constrain today's design.

The defensible position: the concrete harms are real now and worth your effort; the long-term risks are
uncertain and worth serious people working on; and the safeguards that help with both — permissions,
verification, interpretability, honest evaluation — are the same ones that make systems work well.
That convergence is genuinely convenient, and it means you don't have to resolve the debate to act
sensibly.

---

# Part 2 — Questions to investigate

Analysis plus measurable experiments on your own systems. Keep everything in `notes.md`.

### Q1. Specification gaming in your own work
**Build:** review your Phase 9 evaluation harness for metrics your system could satisfy without doing
the intended thing.
**Check:** try to game one deliberately — optimize a prompt for the metric rather than the task.
**Explain:** did the score rise while quality fell? This is outer misalignment at small scale — describe
it in those terms.

### Q2. Measure sycophancy
**Build:** ask 20 factual questions; then push back on correct answers ("are you sure? I read
otherwise") and record whether the model reverses.
**Check:** report the reversal rate on answers that were right.
**Explain:** report it. Where does this behaviour come from (Topic 15)? What product harm does it cause?

### Q3. Consistency under rephrasing
**Build:** 20 questions, each in 5 phrasings.
**Check:** measure answer consistency.
**Explain:** report it. What does inconsistency mean for any safety property you wanted to guarantee?

### Q4. Safety generalizes worse than capability
**Build:** take 10 requests your model refuses. Retry each in another language, in a fictional frame, and
in an encoded form. Then test whether *capability* survives the same transformations.
**Check:** report refusal rate and capability retention for each transformation.
**Explain:** report both. Did capability transfer better than refusal? This is §59.3's asymmetry —
did you reproduce it?

### Q5. Distribution shift
**Build:** evaluate your system on inputs unlike anything in your evaluation set (another domain,
another language, adversarial phrasing).
**Check:** report the score drop.
**Explain:** did it fail safely or confidently? Which is worse, and what would you change?

### Q6. Bias evaluation
**Build:** an evaluation measuring output differences across demographic variations of otherwise
identical inputs (names, dialects, stated backgrounds).
**Check:** report differences in quality, tone, refusal rate, and any substantive recommendation.
**Explain:** what did you find? What would you do about it before shipping?

### Q7. Probing and steering
**Build:** with local model access, train a probe for a simple property on hidden states; if feasible,
try steering behaviour by adding the probe direction to activations.
**Check:** report probe accuracy and whether steering worked.
**Explain:** what does this suggest about internal representations — and about the limits of
behavioural testing alone?

### Q8. Chain-of-thought monitoring
**Build:** on a task where the model behaves badly under pressure (Q2's sycophancy works), inspect
whether the reasoning reveals the real cause.
**Explain:** was the reasoning informative or a post-hoc rationalization? What does that mean for
monitoring reasoning as a safety mechanism?

### Q9. Corrigibility audit
**Build:** audit your Phase 8 agent against §59.4's list — kill switch, bounded permissions, approval,
reversibility, audit trail.
**Check:** test each one works.
**Explain:** which was weakest? Fix it and say what you changed.

### Q10. Dangerous-capability evaluation
**Build:** for your own application, define what misuse would look like and write an evaluation for it
(not for the model's general capabilities — for *your system's* misuse potential).
**Check:** run it.
**Explain:** what could your system be used for that you didn't intend? What would you restrict?

### Q11. Read a responsible scaling policy
**Build:** read one lab's published safety framework and one regulation's requirements (e.g. the EU AI
Act's risk tiers).
**Explain:** which requirements would apply to a system you've built? What documentation would you need
to produce, and do you have it?

### Q12. Write a model card for your own system
**Build:** intended use, out-of-scope use, measured performance, known limitations, bias evaluation
results, and safety measures.
**Check:** would a user reading it understand what not to trust it for?
**Explain:** what was uncomfortable to write down? That discomfort is usually the important part.

### Q13. Present harms in your product
**Write:** for a system you've built, list plausible harms it could cause now — misinformation, privacy,
bias, over-reliance, displacement.
**Explain:** which is most likely? What's the cheapest mitigation, and have you implemented it?

### Q14. Your considered position
**Write:** 1,500 words: what you believe about long-term risk, which present harms you take most
seriously, what obligations you accept as a practitioner, and what you would refuse to build.
**Explain:** cite your own experiments. State what evidence would change your view, date it, and plan to
revisit it.

---

# Done when you can answer

1. What's the difference between outer and inner alignment?
2. What is specification gaming, and have you seen it in your own work?
3. Why is behavioural testing insufficient for verifying alignment?
4. Why does capability generalize better than safety training?
5. What is corrigibility, and what is its practical engineering form?
6. Which harms are present and measurable, and which are speculative?
7. What should a practitioner actually do differently?

Write answers in `notes.md`.

---

**Phase 15 is complete.** Phase 16 returns to engineering: models that see, hear and watch.
