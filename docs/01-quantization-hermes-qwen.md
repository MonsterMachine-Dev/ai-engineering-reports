# From Hermes to Qwen: A Local AI Quantization Journey

**Author:** Nicolas Yammouny (MMC-DEV Lab)
**Duration:** Multi-day session, ~3-4 days of active work
**Hardware:** Consumer laptops, no dedicated GPU

---

## 1. Purpose

The goal was never to train a model from scratch — it was to take existing open-weight
models and **quantize them properly**, tailored to real, constrained consumer hardware,
in service of two concrete goals:

1. Make daily-use local AI models (Hermes for strategy/reasoning, Qwen3-Coder-Next for
   coding/agentic work) run smoothly and efficiently on hardware without a dedicated GPU.
2. Build genuine, hands-on expertise in the local-AI quantization pipeline as a
   differentiator for a future AI router/system product ("Ask Your Way" — see
   [`03-ask-your-way-router.md`](./03-ask-your-way-router.md)).

No training, no fine-tuning datasets, no RLHF — purely **quantization, calibration
(imatrix), and benchmarking**, done rigorously, with real hardware constraints as the
guiding limitation throughout.

---

## 2. Hardware

Three machines were used across this project: one production machine (kept isolated
from experimental workloads), one dedicated AI test bench, and one lower-spec target
device the final lightweight build was validated against. Ambient room temperature
throughout testing was approximately 27°C.

A clean operational split was established mid-project: the production machine stays
production-only; the test bench absorbs all quantization/build work. This resolved a
real crash (`std::terminate()` / uncaught exception) traced to environment differences
between a rolling-release Linux stack and a more standard, predictable one.

---

## 3. Phase One — Hermes-4.3-36B (dense architecture)

### 3.1 Starting point
- Base model: **NousResearch/Hermes-4.3-36B**, built on **ByteDance Seed-OSS-36B-Base**
  (confirmed via GGUF metadata inspection — `general.architecture = seed_oss`)
- Dense architecture, 64 transformer blocks, GPT-2 BPE tokenizer, 155,136-token vocab
- Starting quant on disk: Q6_K (~27.6 GB)

### 3.2 Baseline benchmark

| Quant | Size | pp128 | tg32 |
|---|---|---|---|
| Q6_K (original) | 27.62 GiB | 1.01 t/s | 0.53 t/s |

A thread-count test (`-t 8` vs `-t 4`) showed almost no difference in throughput — the
clear fingerprint of a **memory-bandwidth-bound** workload, not a compute-bound one.
This became a foundational lesson for the rest of the project: on CPU-only inference,
adding threads does not help once you are bandwidth-bound; the only real lever is
**reducing how much data must move through memory per token** — i.e., quantization.

### 3.3 First requantization attempt — quant *type* matters more than quant *size*
Built a small imatrix calibration corpus from real project vocabulary, then requantized
directly from the already-quantized Q6_K file to IQ4_XS and Q4_K_M using
`--allow-requantize` (llama.cpp disables requantizing from an already-lossy source by
default, as a deliberate safety measure).

| Quant | Size | pp128 | tg32 |
|---|---|---|---|
| Q6_K (original) | 27.62 GiB | 1.01 t/s | 0.53 t/s |
| IQ4_XS | 18.15 GiB | 0.76 t/s (**worse**) | 0.54 t/s (flat) |
| Q4_K_M | 20.26 GiB | 1.51 t/s (**+50%**) | 0.73 t/s (**+38%**) |

IQ-format quants pack bits more tightly but cost more compute to decode on CPU. On a
bandwidth-bound workload, that extra decode cost can fully offset the bandwidth savings
from the smaller file. K-quants use simpler, cheaper-to-decode math and, on CPU-only
hardware, frequently outperform IQ-quants of similar or even smaller size. **This was
one of the two or three most important lessons of the entire project.**

### 3.4 The "identical hardware, half the speed" mystery — solved
Running the identical Q4_K_M file on the test bench produced roughly half the tg32
speed of the production machine, despite reportedly similar hardware. Root-caused
through a disciplined, sequential elimination process:

1. **CPU governor** — the test bench was on `powersave`, throttling sustained
   inference regardless of raw hardware. Switching to `performance` was necessary but
   not sufficient.
2. **Background contention** — a separate inference service was silently running in
   the background, holding ~20GB RAM and actively serving an unrelated task mid-benchmark,
   contaminating every reading.
3. **Thermal throttling** — after a full day of sustained builds, the CPU was sitting
   1°C from its critical threshold. Physical cooling brought it back to a safe range.

Once all three were addressed, real generation speed **jumped from 0.36 to 0.9 t/s** —
actually better than the original reference number. The machine was never weaker; it
was simply misconfigured and unmonitored.

**Standing lesson, applied for the rest of the project:** before trusting any benchmark,
always check, in order: (1) CPU governor, (2) background processes competing for
RAM/CPU, (3) swap usage, (4) thermal state. A "slow model" is very often none of those
three things about the model itself.

### 3.5 The correct pipeline — clean source, not compounded lossy steps
Once it was clear that requantizing from an already-lossy Q6_K file put a ceiling on
achievable quality, the project pivoted:

1. Download the full BF16/FP16 safetensors source directly from HuggingFace (~68GB,
   14 shards).
2. Convert to a clean F16 GGUF (~68GB output).
3. Build a larger, more diverse imatrix corpus (~1,800 words) covering real usage
   patterns.
4. Run `llama-imatrix` against the clean F16 source — not the lossy quant.
5. Quantize once, directly, from F16 to Q4_K_M — no `--allow-requantize` needed, since
   this is a single lossy step rather than two compounded ones.

**Result:** a legitimately higher-quality Q4_K_M build (21GB), validated live via
`llama-cli`, producing coherent, well-structured strategic output at real-world speed
once the governor/thermal/contention issues above were resolved.

---

## 4. Phase Two — Qwen3-Coder-Next (80B-parameter MoE)

### 4.1 A critical MoE-specific calibration finding
Early imatrix runs against a small corpus showed many `ffn_*_exps` tensors with only
60-95% "partial data" coverage in the `save_imatrix` warnings. With 512 experts and only
10 active per token, a short/narrow calibration corpus leaves the majority of experts
poorly or entirely uncalibrated. **MoE models require substantially larger and more
diverse imatrix corpora than dense models of similar size** to achieve meaningful
expert coverage — a lesson that did not apply to Hermes (dense) in the same way.

To address this, the calibration corpus was expanded to a combined **~340,000-word /
24MB corpus**, merging hand-written text covering real project domains, a sample of
real multi-language coding Q&A data, and real agentic tool-calling examples.

### 4.2 The correct pipeline for Coder-Next
1. Downloaded the full BF16 safetensors — **159GB**, 40 shards.
2. Converted to F16 GGUF (**149GB**).
3. Ran `llama-imatrix` against the clean F16 source with the full corpus.
   - **A critical scoping error occurred here:** the first attempt used the entire
     24MB corpus (11,578 chunks) against the *full-precision* F16 model, producing an
     estimated ETA of **765 hours (32 days)**. The corpus was cut to a 233-chunk
     sample; the run then completed in **~24 hours**.
4. Quantized directly from the clean F16 source using the properly-calibrated imatrix:

| Quant (from clean F16 + real imatrix) | Size | Quantize time |
|---|---|---|
| Q3_K_M | 36 GB | ~50 minutes |
| IQ2_M | 25 GB | (first attempt corrupted by a power interruption mid-write; rerun succeeded) |
| Q4_K_M | 46 GB | ~84 minutes |

### 4.3 A real corruption incident and its resolution
An unplanned power interruption during an IQ2_M quantize left a file that appeared
complete by size but failed to load (`invalid magic characters: '????', expected
'GGUF'`) — the header had not been fully written. This was correctly diagnosed (not
assumed) by attempting to load the file directly, deleting it, and rerunning the
quantize from the still-intact clean F16 source and imatrix. The rerun completed
cleanly.

**Lesson:** a file matching an "expected" size is not proof of a valid file, especially
after any unplanned interruption. Always attempt to load a freshly-built model file
before trusting it.

### 4.4 Final head-to-head: self-built quant vs. a professional reference quant

| | Self-built (own imatrix, clean F16) | Professional reference build |
|---|---|---|
| Size | 45.15 GiB | 45.49 GiB |
| pp128 | 4.87 ± 0.70 | 4.42 ± 0.63 |
| tg32 | 1.22 ± 0.13 | 1.14 ± 0.13 |

The self-built quant was **smaller and consistently faster** on both metrics
(**+10% prompt-processing, +7% generation**) — a genuine, repeatable win over a
professionally-built dynamic quant, using an imatrix calibrated entirely from real
project data.

Two real-world reasoning prompts were also run identically against both models: both
made the same wrong assumption when given an ambiguous project codename without
tech-stack context (a shared blind spot in the underlying base model, not something
introduced or fixed by either calibration), and both reasoned correctly and specifically
once given full technical context — **prompt specificity had a larger effect on answer
quality than the choice of quant.**

---

## 5. Summary of Durable Technical Lessons

1. **Quant type matters more than quant size.** K-quants decode more cheaply than
   IQ-quants on CPU; a smaller IQ file can be slower than a larger K-quant file.
2. **MoE beats dense on CPU, regardless of total parameter count**, because only a
   fraction of experts activate per token.
3. **Never requantize from an already-quantized source if a clean, full-precision
   source is obtainable.** Compounded lossy compression sets a lower quality ceiling
   than a single clean quantization step.
4. **MoE models need dramatically larger, more diverse imatrix calibration corpora**
   than dense models of similar size, to achieve meaningful coverage across all
   experts.
5. **Always rule out environment before blaming the model or the quantization:**
   CPU governor, background process contention, swap usage, thermal throttling —
   in that order — before trusting a slow number.
6. **A file matching its expected size is not proof of validity**, especially after
   any power interruption.
7. **Prompt specificity matters as much as model quality.**
8. **A dedicated, cooled, monitored test machine should be kept separate from any
   production machine** — resource contention makes both unreliable to reason about.

---

## 6. Closing Note

This project began with genuine hesitation about running a small local model a year
prior, and closed with a self-calibrated, self-benchmarked 80B-parameter MoE model
outperforming a professionally-built quantization on the same hardware — built entirely
through direct experimentation, careful diagnosis of real failures (power outages,
thermal throttling, silent background contention, file corruption, a 32-day ETA
miscalculation), and a disciplined refusal to accept an unexplained result without
first ruling out the environment.
