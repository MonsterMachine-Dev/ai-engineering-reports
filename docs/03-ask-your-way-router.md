# Ask Your Way (AYW) — A Local, Offline Multi-Model AI Router

**Author:** Nicolas Yammouny (MMC-DEV Lab)

---

## What It Is

Ask Your Way is an offline AI router that runs entirely on local hardware — no cloud
API calls, no third-party inference dependency. It sits in front of several local
language models and directs each incoming request to the model best suited to the
task, rather than forcing every query through a single general-purpose model.

The philosophy behind it is the same one behind the rest of the ecosystem it serves:
privacy shouldn't be a checkbox you have to trust someone else on. Running the models
locally isn't a cost-saving measure — it's the actual point.

---

## Architecture

- **Multi-model routing**, serving requests across several purpose-selected local
  LLMs rather than one generalist model:
  - A fast, always-on model for lightweight/quick requests
  - A larger reasoning/creative model for strategy and long-form work
  - A dedicated coding/agentic model for development tasks
  - A dedicated legal/reasoning-focused model
- **Per-response performance monitoring** in the frontend — tokens/second, token
  count, elapsed time, and which model handled the request, surfaced live rather than
  hidden behind a generic loading spinner.
- **Independent operation** — AYW runs as its own service, with no shared startup
  dependency on any other product in the ecosystem, so it can be developed, restarted,
  and iterated on without affecting anything else running alongside it.

---

## Role as a Testing Ground

AYW also functions as the proving ground for new models before they're promoted to
other products in the ecosystem — new quantized builds (see
[`01-quantization-hermes-qwen.md`](./01-quantization-hermes-qwen.md)) and new
multimodal backends (see
[`02-multimodal-gemma4-agent.md`](./02-multimodal-gemma4-agent.md)) are both validated
here first, on real usage patterns, before being wired into any production-facing
product.

---

## Why This Matters

Running a single model well is a solved problem. Running several models well — each
selected deliberately for what it's actually good at, monitored for real performance
rather than assumed, and orchestrated without a single point of cloud dependency — is
the harder, more interesting problem, and it's the one AYW is built to solve.
