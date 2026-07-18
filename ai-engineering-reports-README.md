# AI Engineering Reports — MMC-DEV Lab

Technical write-ups from hands-on local AI infrastructure work: quantization,
benchmarking, multimodal model integration, and the routing system that ties them
together. No training or fine-tuning here — this is deployment, calibration, and
systematic debugging of real open-weight models on real, constrained consumer
hardware, with no dedicated GPU.

Author: **Nicolas Yammouny**, founder of MMC-DEV Lab

- [github.com/MonsterMachine-Dev](https://github.com/MonsterMachine-Dev)
- [huggingface.co/Nicolasy1979](https://huggingface.co/Nicolasy1979)

---

## Reports

1. **[From Hermes to Qwen: A Local AI Quantization Journey](docs/01-quantization-hermes-qwen.md)**
   Quantizing and benchmarking Hermes-4.3-36B (dense) and Qwen3-Coder-Next (80B-parameter
   MoE) from clean full-precision sources. The self-built quant outperformed a
   professionally-built reference build by +10% prompt-processing and +7% generation
   speed on identical hardware. Covers CPU-bound performance diagnosis, MoE-specific
   calibration requirements, and recovering from real hardware/power failures mid-pipeline.

2. **[Building a Multimodal Marketing AI Agent on Gemma 4](docs/02-multimodal-gemma4-agent.md)**
   Building a text + vision + voice AI backend on Google's Gemma 4 model family,
   released only weeks prior. Covers real conversion bugs on a brand-new architecture,
   a multimodal projector misconfiguration, and two cases where an initial diagnosis of
   "broken" was wrong — reversed by a more concrete test.

3. **[Ask Your Way (AYW) — A Local, Offline Multi-Model AI Router](docs/03-ask-your-way-router.md)**
   The routing architecture that serves as the deployment target and testing ground
   for the models covered in the two reports above.

4. **[Cook Your Way (CYW) — An Offline-First, Inventory-Aware Cooking App](docs/04-cook-your-way.md)**
   Architecture and stack for a recipe/inventory app built around offline-first
   design and food-type-aware expiry logic.

---

## Why This Exists

These are real engineering logs, not marketing copy — including the dead ends, the
wrong assumptions, and the incidents that didn't resolve cleanly on the first try.
That's deliberate: the debugging process is the actual skill on display here, not just
the final benchmark numbers.

## License

MIT — see [LICENSE](LICENSE).
