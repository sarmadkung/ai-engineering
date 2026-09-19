# Topic 61 — Audio

**Why this topic:** speech is the interface people actually want, and audio is where a lot of business
information lives — calls, meetings, voice notes. This topic is transcription, synthesis, and the
engineering of a voice application, where **latency is the whole product**.

---

# Part 1 — Theory

## 61.1 Speech recognition (ASR)

Audio in, text out. Modern systems (Whisper and its successors, plus provider APIs) are transformer
encoder-decoders trained on very large amounts of audio, and they're good enough that transcription is
rarely the hard part any more.

What still matters in practice:

- **Word error rate (WER)** varies enormously by accent, language, domain vocabulary, and audio quality.
  Report it on *your* audio, not from a paper.
- **Domain vocabulary** — product names, drug names, jargon and acronyms are where errors cluster. Fixes:
  prompt/bias the model with expected terms where supported, or post-process against a vocabulary list.
- **Diarization** — who spoke when. Usually a separate model or service, and noticeably less reliable
  than transcription itself, especially with overlapping speech.
- **Timestamps** — word or segment level, needed for search, subtitles, and linking a quote back to the
  audio.
- **Hallucination on silence** — ASR models can invent text during long silences or noise (Whisper is
  known for this). Detect and strip it, or your transcripts contain fabricated sentences.
- **Long audio** — chunk with overlap and stitch, handling words split across boundaries. Same problem
  shape as Topic 22's chunking.
- **Streaming vs batch** — streaming gives partial results with lower accuracy and revisions; batch is
  more accurate and arrives all at once.

## 61.2 Text-to-speech (TTS)

Text in, audio out. Current neural TTS is close enough to natural that quality is rarely the
constraint; the constraints are latency, cost and control.

What to know: **voice cloning** from a short sample exists and raises obvious consent and impersonation
issues that are your responsibility, not the vendor's; **prosody control** (emphasis, pauses, emotion)
varies by provider and is the main quality differentiator; **streaming synthesis** lets audio start
before the full text is ready, which is essential for conversation; and **pronunciation** of names,
acronyms and numbers usually needs explicit handling.

## 61.3 Audio understanding beyond words

Speech-to-text discards information: tone, emotion, hesitation, background sound, music, and who is
talking. Native audio models (models that take audio tokens directly rather than a transcript) can use
it — which matters for sentiment on support calls, detecting confusion or distress, and any task where
*how* something was said carries the meaning.

The trade: transcription is cheap, inspectable and searchable; native audio preserves information but is
costlier and harder to debug. Most systems transcribe, and reach for native audio when the paralinguistic
content is the point.

## 61.4 The voice pipeline, and why latency dominates

```
user speaks -> VAD detects end of speech -> ASR -> LLM -> TTS -> audio plays
```

Human conversation tolerates roughly 200–500 ms of silence before it feels broken. Your budget:

```
VAD/end-of-speech detection   100-300 ms
ASR                           100-500 ms
LLM first token               300-1000 ms
TTS first audio               100-300 ms
network                        50-200 ms
```

That adds to well over a second naively, which feels sluggish. So the engineering is all about
overlapping stages:

- **Stream everything.** Transcribe as they speak; send text to the LLM as soon as the utterance ends;
  synthesize the first sentence while the model writes the second.
- **Sentence-level TTS** — don't wait for the whole answer.
- **Speculative start** — begin processing before you're certain the user has finished, and discard if
  they continue.
- **Interruption handling** — the user talks over the assistant, which must stop immediately, discard
  queued audio, and treat the interruption as input. Getting this right is what separates a natural
  voice agent from a frustrating one.
- **Shorter answers.** A voice assistant's answers must be much shorter than a chat assistant's, both
  for latency and because listening is linear.

**Speech-to-speech models** (audio in, audio out, no transcription in between) collapse the pipeline,
cutting latency and preserving tone, at the cost of controllability and inspectability — you lose the
text you were logging, filtering and grounding against.

## 61.5 Engineering concerns

- **Cost** — usually priced per minute or per character; a long call is a real expense, and TTS of long
  answers adds up.
- **Storage and privacy** — recordings are personal data, often subject to consent requirements and
  two-party consent laws. Decide retention before you build.
- **Injection through speech** — spoken instructions are untrusted input exactly like text, and they
  bypass text-only filtering (Topic 41).
- **Accessibility** — voice interfaces both help and exclude; provide a text path.
- **Error handling in voice** — you cannot show a stack trace. Mis-hearings need graceful confirmation
  ("did you mean...?") rather than confident wrong action, especially before any consequential step.

---

# Part 2 — Questions to implement

Build `audio/` here. Use a hosted ASR/TTS API, or run Whisper locally. You need real audio: your own
voice, an accented sample, a noisy recording, a multi-speaker conversation, and something with domain
jargon.

### Q1. Transcribe and measure
**Build:** transcribe 10 clips and compute WER against hand-written ground truth.
**Check:** report WER per clip.
**Explain:** report your overall WER. Which clip was worst, and what characterized it?

### Q2. Accent, noise, and domain vocabulary
**Build:** transcribe clean, accented, noisy, and jargon-heavy audio.
**Check:** report WER per category.
**Explain:** where do errors cluster? What fraction were domain terms?

### Q3. Vocabulary biasing
**Build:** supply expected terms (via prompting/biasing if supported, else post-process against a term
list with fuzzy matching).
**Check:** re-measure WER on the jargon clip.
**Explain:** report the improvement. What did post-processing get wrong?

### Q4. Hallucination on silence
**Build:** transcribe clips with long silences, background music, and non-speech noise.
**Check:** look for invented text.
**Explain:** did it fabricate? How would you detect and strip this automatically?

### Q5. Long audio chunking
**Build:** transcribe a 30+ minute recording by chunking with overlap and stitching.
**Check:** inspect the boundaries for duplicated or lost words.
**Explain:** what overlap did you need? How did you dedupe the seams?

### Q6. Diarization
**Build:** diarize a multi-speaker conversation.
**Check:** measure speaker-attribution accuracy against ground truth.
**Explain:** report it. Where did it fail — overlapping speech, similar voices, short turns?

### Q7. Streaming vs batch ASR
**Build:** transcribe the same audio both ways.
**Check:** compare WER and time-to-first-text.
**Explain:** report both. Which would you use for a voice assistant, and which for meeting notes?

### Q8. TTS quality and control
**Build:** synthesize the same text with different voices and prosody settings, including text with
names, numbers, acronyms and abbreviations.
**Check:** listen and note every mispronunciation.
**Explain:** what needed explicit handling? Measure time-to-first-audio.

### Q9. The naive voice loop
**Build:** ASR → LLM → TTS with no overlapping, measuring each stage.
**Check:** report the latency breakdown and total.
**Explain:** report your total. How does it feel to use? Which stage dominates?

### Q10. The streamed voice loop
**Build:** streaming ASR, LLM streaming, sentence-level TTS, all overlapped.
**Check:** measure total perceived latency (user stops speaking → first audio).
**Explain:** report before and after. Which optimization helped most?

### Q11. Interruption
**Build:** interruption handling — stop playback immediately, discard queued audio, treat the interruption
as input, and keep the conversation state consistent.
**Check:** interrupt mid-sentence repeatedly.
**Explain:** what broke first? Why is this so important to conversational feel?

### Q12. Answer length for voice
**Build:** the same assistant with chat-length answers and with voice-appropriate answers (2–3 sentences,
no lists).
**Check:** compare total interaction time and your own experience of using each.
**Explain:** what prompt changes were needed, and what did you have to give up?

### Q13. Speech-to-speech comparison
**Build:** if you have access to a speech-to-speech model, run the same interactions through it and your
pipeline.
**Check:** compare latency, naturalness, cost, and what you can log and filter.
**Explain:** report all four. What did you lose by removing the text layer? Would you accept that?

### Q14. Safety and privacy in voice
**Build:** a spoken prompt-injection attempt, plus a consequential action requiring confirmation, plus a
documented retention policy with real deletion.
**Check:** did the injection work? Does a mis-heard command get confirmed before acting? Does deletion
remove the audio and the transcript?
**Explain:** which control was missing before you added it?

---

# Done when you can answer

1. What determines transcription accuracy, and where do errors cluster?
2. Why do ASR models hallucinate on silence, and how do you handle it?
3. What information does transcription discard, and when does that matter?
4. What's the latency budget for natural conversation, and how do you meet it?
5. Why is interruption handling so important?
6. What do speech-to-speech models gain and lose?
7. Why must voice interfaces confirm before consequential actions?

Write answers in `notes.md`.
