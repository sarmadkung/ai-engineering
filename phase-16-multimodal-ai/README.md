# Phase 16 — Multimodal AI

**Goal:** work with images, audio and video. This is where the document problems from Phase 5 finally get
solved, and where voice and computer-use agents become possible.

---

## What I will do

| # | Topic | What I build | Status |
|---|---|---|---|
| 60 | [Vision](60-vision/) | `vision/` — document extraction cascade, structured output, multimodal retrieval | ready |
| 61 | [Audio](61-audio/) | `audio/` — transcription, TTS, a streamed voice loop | ready |
| 62 | [Video](62-video/) | `video/` — sampling strategies, a computer-use agent | ready |

## Which things to learn

**60. Vision** — how an image becomes patch tokens the language model attends over; why images cost
tokens and resolution is a cost dial; what VLMs are reliable at (documents, charts, screenshots, forms)
and what they must never be trusted for (counting, precise spatial relations, exact measurement); OCR
engine versus model, and using both; **a cascade** — cheap extraction first, vision only on failure;
vision embeddings for multimodal retrieval; and injection through text inside an image, which bypasses
your text filters entirely.

**61. Audio** — ASR accuracy and where errors cluster (accents, jargon, noise); hallucination on silence;
diarization; chunking long audio; TTS control and voice-cloning consent; what transcription discards and
when that matters; **the latency budget for natural conversation** and how overlapping stages meets it;
interruption handling as the difference between natural and frustrating; speech-to-speech and what you
lose with it; confirming before consequential action when you might have mis-heard.

**62. Video** — why every approach is a **sampling strategy**; uniform, keyframe, adaptive and hierarchical
sampling; temporal reasoning and its limits; **audio plus keyframes as a strong, cheap baseline**; the
cheap-detector/expensive-interpreter pattern; multimodal agents and computer use, where grounding and
verification (not reasoning) are the limiting factors; and the safety requirements of an agent holding a
mouse.

## Prerequisites

Phase 4 for the API mechanics, Phase 5 for retrieval, Phase 6/7 for the computer-use agent. Real
material: scanned PDFs, screenshots, charts, your own voice recordings, a long meeting recording, and a
screen recording. `ffmpeg` for frames.

**Next:** Phase 17 — the ten projects that turn all of this into a portfolio.
