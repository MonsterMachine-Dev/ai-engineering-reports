# Building a Multimodal Marketing AI Agent on Gemma 4

**Author:** Nicolas Yammouny (MMC-DEV Lab)
**Duration:** Single session, ~15 working hours
**Task:** Building and validating the model backend for a marketing-focused AI agent
prototype, for a non-technical business stakeholder to test.

---

## 1. Purpose

Where the first project (see
[`01-quantization-hermes-qwen.md`](./01-quantization-hermes-qwen.md)) was about
squeezing maximum quality and speed out of text-only coding/reasoning models, this
session had a different shape entirely: **build a working multimodal AI backend** —
text, image understanding, and voice — for a real, non-technical end user to test as a
V1 prototype, explicitly understood as an early, upgradable draft rather than a
finished product.

Two models were targeted from Google's newly-released Gemma 4 family:
1. **Gemma-4-26B-A4B-it** — MoE architecture, 4B active parameters, text-only.
   Intended as the fast, general-purpose chat/strategy tier.
2. **Gemma-4-12B-it** — dense, `gemma4_unified` architecture, natively multimodal
   (text + vision + audio in one set of weights, no separate encoder model). Intended
   as the vision/voice tier — reading documents and images, transcribing and
   responding to voice notes.

No training or fine-tuning — same as before, purely
**download → convert → calibrate → quantize → validate**, but this time with the added
complexity of a genuinely new, still-unstable multimodal architecture, released only
weeks prior.

---

## 2. Model Acquisition

- **Gemma-4-26B-A4B-it**: one network drop occurred mid-transfer (~30GB in); the
  download tool's resume behavior worked cleanly, resuming from the exact byte offset
  rather than restarting.
- **Gemma-4-12B-it**: initially believed unnecessary, until it became clear the
  26B-A4B variant is **vision-only, with no audio support at all** — audio is limited
  to the smaller Gemma 4 variants. This corrected the model selection mid-session:
  12B-it was pulled specifically because it's the smallest Gemma 4 variant with
  **both** native vision and audio.
- Disk space discipline carried over from the prior project: cleanup of stale files
  from earlier work freed the headroom needed before pulling ~75GB combined.

---

## 3. Conversion — A New, Immature Architecture Surfaces Real Bugs

### 3.1 Tokenizer failure on first conversion attempt
Conversion completed tensor export cleanly for 26B-A4B but crashed at the final
vocabulary-writing step (`AttributeError: 'list' object has no attribute 'keys'`).
Root cause: the installed `transformers` package was too old to natively recognize the
brand-new `gemma4` architecture. Fixed with a package upgrade inside the working venv.

### 3.2 A corrected misconception: corpus size, not MoE-vs-dense, drives imatrix time
Early in the session, a false pattern was proposed: that dense models handle large
imatrix corpora fine while MoE models don't. Testing corrected this directly — the
real variable was corpus **size** (chunk count), not architecture. A properly small,
purpose-built corpus resolves long imatrix runs on any model, dense or MoE.

### 3.3 A second correction: MoE actually has a compute advantage, not a disadvantage
Per-token compute for MoE models is proportional to *active* parameters, while dense
models compute across *all* parameters every token. The uniform slowness observed
across both Gemma models during testing was traced instead to running unquantized
F16 — a memory-bandwidth-bound cost independent of architecture, consistent with the
lessons from the prior project.

---

## 4. Calibration & Quantization

### 4.1 26B-A4B — a deliberate "rough cut" strategy
Given the explicit goal of getting a testable build out same-day, a decision was made
to skip full imatrix calibration for an initial pass, then empirically validate on the
actual target hardware rather than assume:

| Quant | Generation speed | Swap usage | Verdict |
|---|---|---|---|
| Q4_K_M | ~2.69 t/s | 3.1–5.2GB (actively swapping) | Poor fit |
| Q3_K_M | ~3.24 t/s | ~850MB (negligible) | **Standardized on this** |

Q3_K_M was faster *and* avoided swap entirely — a clean, evidence-based
standardization rather than a guess.

### 4.2 12B-it — a small, purpose-built corpus
Rather than reuse a large generic corpus, a small (~3.5KB), domain-specific
calibration corpus was hand-written — covering document reading, voice-memo-style
transcripts, and image-description prompts matching the actual intended use case.
Quantized to Q4_K_M (6.9GB) — no Q3 variant was needed, since Q4 already left ample
headroom on the target hardware.

---

## 5. Multimodal Validation — The Real Work of This Session

This is where the session diverged most from the prior, purely-text project, and
where the most real debugging happened.

### 5.1 The mmproj correction
Initial assumption (later corrected): that 12B-it's "encoder-free" architecture needed
no separate multimodal projector file. This was wrong — llama.cpp's GGUF format still
requires a companion **mmproj** file for any multimodal input, even on unified
architectures. Confirmed directly via an HTTP 500 error pointing at the missing file.

### 5.2 Audio — three failures, then a corrected diagnosis
1. First test (vague prompt: "transcribe this audio"): failed, model hallucinated
   unrelated content.
2. Second test (different audio format, same vague prompt): failed identically —
   ruling out format as the cause.
3. **Third test — a clear, concrete factual question, spoken aloud:** succeeded
   completely — correct transcription, correct answer, correct language, and notably
   **faster** (~162 seconds vs. 8–10 minutes) because the response was short and
   direct rather than a long uncertain ramble. The same model also correctly
   understood a question asked in colloquial Lebanese Arabic.
4. Fourth test — a real, open-ended marketing question: succeeded with a detailed,
   accurate, well-structured response.

**Corrected conclusion:** the first two failures were caused by prompt vagueness
giving the model nothing concrete to anchor to, not a broken audio pipeline.

### 5.3 Vision — a genuine architecture-ambiguity dead end, then a full success
A metadata inspection of the mmproj file showed zero vision transformer blocks —
apparently confirming no real vision layers were exported. External research surfaced
a genuine, unresolved disagreement in the community about whether this architecture's
vision tower was even supported yet by the conversion tooling.

A first HTTP-based vision test failed. A comparison test against a different vision
backend also produced a wildly wrong description — traced to a separate, unrelated
bug: the test image file was actually a mislabeled WebP file with a `.png` extension,
causing a false negative unrelated to either model's real capability.

The actual breakthrough came from bypassing the HTTP layer and testing via the
model's own CLI tool directly:
- First CLI attempt **crashed**, revealing the true root cause of *all* the earlier
  vision failures: a **chat template handling gap**, not a missing-capability problem.
- With the correct template flag added and the corrected (real) PNG file, the CLI
  test succeeded outright — an accurate, detailed description of the real test image.
- The same fix applied to the HTTP server produced a final, complete success: the
  model correctly read a real dashboard screenshot end-to-end via the actual API path
  the product will use in production, identifying specific UI details and producing a
  genuinely sharp, specific critique — not generic filler.

**Corrected conclusion:** the misleading metadata field was not the real signal;
vision (like audio) does work on this architecture. The true blocker throughout was
chat template handling, resolved by a single flag.

---

## 6. Router Integration Planning

An existing multi-model router (already serving several text and vision models) was
reviewed as the target integration point for the newly-validated Gemma 4 models — a
dedicated marketing-model slot existed in the router's structure, unused until now, to
be filled with the 26B-A4B build; the existing vision backend is planned to be
replaced by Gemma-4-12B-it, unifying vision and voice into a single backend rather
than two separate models.

---

## 7. Summary of Durable Lessons

1. **A confidently-stated diagnosis is not the same as a confirmed one.** Twice this
   session, native audio and native vision were prematurely judged broken based on
   incomplete evidence — both times, insisting on a clearer, more concrete test
   reversed the conclusion. Arguably the single most important process lesson of the
   session.
2. **Vague prompts produce vague, sometimes hallucinated, results — especially with
   audio.**
3. **Corpus size — not model architecture — is what drives imatrix calibration time.**
4. **A file's extension is not proof of its format.** Always verify with a real file
   inspection tool, not the filename, when a result looks wrong.
5. **Chat template handling can silently break multimodal input without any
   indication that capability itself is the problem.**
6. **Empirical hardware validation beats assumption every time.**
7. **A brand-new model architecture (weeks old) will have real, undocumented rough
   edges** — worth budgeting real debugging time for, separate from the "normal"
   quantization pipeline work.

---

## 8. Closing Note

Where the first project closed with a clean, unambiguous technical win, this session
closed messier and, in some ways, more honestly: real dead ends, a couple of premature
"this doesn't work" calls that turned out wrong, and a final result that only fully
validated in the last hour of a 15-hour day. But by the end, all three modalities the
product needs — text, vision, and voice — were confirmed working end-to-end, on real
inputs, through the actual API path the finished product will use.
