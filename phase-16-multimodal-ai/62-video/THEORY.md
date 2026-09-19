# Topic 62 — Video

**Why this topic:** video is the hardest modality — images plus time plus audio, at enormous token cost.
It's also where computer-use agents live, since driving a UI means perceiving a changing screen. This
closes the roadmap's technical content.

---

# Part 1 — Theory

## 62.1 Why video is hard

A video is a sequence of images. At 30 frames per second, a one-minute clip is 1,800 images. If each
image costs ~1,000 tokens (Topic 60), that's 1.8 million tokens for a minute — more than most context
windows and unaffordable regardless.

So **every video approach is a sampling strategy**. The question is always: which frames, at what
resolution, and what do you throw away?

- **Uniform sampling** — every Nth frame. Simple; misses events between samples.
- **Keyframe/scene-change sampling** — sample where the content changes. Far more efficient, and the
  usual default.
- **Adaptive sampling** — dense around detected activity, sparse elsewhere.
- **Hierarchical** — coarse pass over the whole video, then dense sampling on the relevant segment. This
  is how you handle long videos, and it's the map-reduce pattern from Topic 19 again.

Providers expose this differently (some accept a video file and sample internally, some expect frames),
but the underlying trade never goes away.

## 62.2 Temporal reasoning

The capability video adds over images: understanding **change and sequence** — what happened, in what
order, what caused what, and what state things are in now.

Current models are decent at: describing what happens in a short clip, identifying an action, answering
questions about a clearly visible event, and summarizing a screen recording.

They're unreliable at: precise timing, counting repeated actions, long-range causality across minutes,
tracking a specific object through occlusion, and anything requiring frame-accurate localization. The
underlying reason is that sampled frames are a lossy, discontinuous view of continuous motion — the
model is reasoning over a slideshow, not a video.

Practical consequence: ask for *what happened* and accept approximate *when*. If exact timing matters,
use a purpose-built detection model or process at high frame rates over a short window.

## 62.3 Audio plus video

Most video carries audio, and for many tasks the audio is the more information-dense channel — a
lecture, a meeting, a demo. A cheap and strong baseline for "understand this video" is:

```
transcribe the audio (Topic 61)  +  sample a few keyframes (Topic 60)  ->  reason over both
```

That combination answers a surprising fraction of real questions at a fraction of the cost of dense
frame analysis. Reach for heavy visual processing only when the visual content is the point.

Alignment matters: keep timestamps on both transcript and frames so you can link a claim to a moment.

## 62.4 Real applications

- **Video understanding and summarization** — meetings, lectures, calls.
- **Screen recordings** — documenting what a user did, generating bug reports, building test cases.
- **Surveillance/monitoring** — usually a small detection model for triggering, a VLM for describing the
  triggered event. Cheap trigger, expensive interpretation.
- **Content moderation** — sampling plus classification.
- **Instructional extraction** — turning a how-to video into written steps.

Notice the recurring architecture: a cheap detector decides *when* to spend, and an expensive model
decides *what it means*. Same cascade as Topic 60.

## 62.5 Multimodal agents and computer use

An agent that perceives a screen and acts on it — the most demanding application, and a real product
category now.

```
screenshot -> model decides an action -> click/type/scroll -> new screenshot -> repeat
```

This is Topic 29's agent loop with pixels as observations. What's specific to it:

- **Cost and latency** — a screenshot per step, and steps are seconds. A 30-step task is expensive and
  slow.
- **Grounding** — converting "click the Submit button" into coordinates. Genuinely hard, and where most
  failures come from. Accessibility trees or DOM (where available) are far more reliable than pixel
  coordinates, which is why browser automation with DOM access beats pure vision (Topic 27).
- **Verification** — did the action do what was intended? The next screenshot is your only feedback, and
  interpreting it is another model call. Verification is both essential (Topic 36) and expensive here.
- **Compounding errors** — a misclick leads to an unexpected state, and recovery requires recognizing
  that state. This is Topic 1's error compounding at its worst.
- **Safety** — an agent with mouse and keyboard has broad, ungated power. Everything in Phase 10
  applies, with sandboxing and approval mandatory rather than advisable.

Honest assessment: usable for narrow, repeatable tasks in controlled environments; unreliable for
open-ended work on arbitrary interfaces. Improving quickly, and the limiting factor is grounding and
verification, not reasoning.

## 62.6 Engineering concerns

- **Cost management** — sample aggressively, downscale, cache frame analyses by hash, and use audio
  first.
- **Storage and bandwidth** — video is large; keep it in object storage and process asynchronously
  (Topic 47's queues).
- **Latency** — video analysis is a background job, not a request handler.
- **Privacy** — video contains people, screens and documents. The strictest obligations of any modality.
- **Untrusted content** — frames containing text can inject instructions, exactly as images can
  (Topic 60).

---

# Part 2 — Questions to implement

Build `video/` here. You need: a short clip with clear action, a long recording (20+ min lecture or
meeting), a screen recording, and a clip where timing matters. Use a video-capable API model, plus
`ffmpeg` for frame extraction.

### Q1. The token arithmetic
**Build:** compute the token cost of a 1-minute video at 30, 5, 1, and 0.2 frames per second, at two
resolutions.
**Check:** validate one configuration by actually sending it.
**Explain:** report the table. Which configurations are affordable at scale?

### Q2. Uniform sampling baseline
**Build:** extract frames uniformly from a short clip and ask what happens in it.
**Check:** compare with your own description.
**Explain:** at what sampling rate did the answer become correct? What did it miss below that?

### Q3. Keyframe sampling
**Build:** scene-change detection to select frames.
**Check:** compare frame count, cost, and answer quality against uniform sampling at similar cost.
**Explain:** report both. How much efficiency did keyframing buy?

### Q4. The missed event
**Build:** a clip containing a brief event (under a second). Sample at 1 fps and at 10 fps.
**Check:** was the event detected in each case?
**Explain:** what does this mean for using sampled video in anything that must not miss things?

### Q5. Temporal ordering
**Build:** 10 questions about the order of events in clips.
**Check:** report accuracy.
**Explain:** where did ordering fail? Did it confuse sequence or invent it?

### Q6. Timing precision
**Build:** ask *when* events happened, and compare against ground truth timestamps.
**Check:** report the error distribution.
**Explain:** how precise was it? Restate §62.2's rule with your data.

### Q7. Counting repeated actions
**Build:** clips with a repeated action (3, 8, 20 repetitions).
**Check:** report counting accuracy.
**Explain:** where did it break down? Compare with Topic 60's object-counting result.

### Q8. Audio-first baseline
**Build:** for the long recording, answer 10 questions using (a) transcript only, (b) frames only,
(c) both.
**Check:** report accuracy and cost for each.
**Explain:** report the table. Which channel carried the information? What does that suggest as a default
architecture?

### Q9. Hierarchical processing of long video
**Build:** coarse pass over the whole recording to locate relevant segments, then dense analysis of the
selected segment.
**Check:** compare accuracy and cost against uniform sampling of the whole thing.
**Explain:** report both. What fraction of the video did you end up analysing densely?

### Q10. Screen recording to documentation
**Build:** turn a screen recording into written step-by-step instructions.
**Check:** follow your own generated instructions and see if they work.
**Explain:** what was wrong or missing? Why is *this* the honest test?

### Q11. Computer-use loop
**Build:** a minimal screenshot → action → screenshot loop on a simple task in a sandboxed environment,
with a step cap.
**Check:** report success rate over 10 attempts, plus steps, cost and latency per attempt.
**Explain:** report all of it. Where did it fail most — perception, grounding, or verification?

### Q12. Grounding: pixels vs structure
**Build:** the same task driven by screenshot coordinates and by DOM/accessibility-tree selectors
(Topic 27).
**Check:** compare success rate, steps, and cost.
**Explain:** report both. Why is structured access so much more reliable, and when do you have no choice
but pixels?

### Q13. Error recovery
**Build:** deliberately inject a wrong click mid-task.
**Check:** does the agent notice the unexpected state and recover?
**Explain:** what happened? What verification step would have caught it sooner (Topic 36)?

### Q14. Cost, safety and a verdict
**Build:** add frame caching by hash, aggressive downscaling, sandboxing, and approval gates for
consequential actions. Then run your computer-use agent on 5 varied tasks.
**Check:** report success rate, cost per successful task, and which safety controls fired.
**Explain:** would you ship this? For which tasks, and with what supervision? Be specific about what
would have to improve.

---

# Done when you can answer

1. Why is every video approach a sampling strategy?
2. What are the sampling strategies, and when does each fit?
3. What are models reliable at temporally, and what not?
4. Why is audio-plus-keyframes such a strong baseline?
5. What is the recurring cheap-detector/expensive-interpreter pattern for?
6. In computer use, what usually fails — reasoning or grounding?
7. Why is verification both essential and expensive in a computer-use loop?

Write answers in `notes.md`.

---

**Phase 16 is complete — and with it, all 62 topics.** Phase 17 is where you build the ten projects
that turn this into a portfolio.
