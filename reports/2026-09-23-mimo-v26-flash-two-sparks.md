# MiMo-V2.6-Flash-RL on two DGX Sparks (day-zero recipe, full agent grid)

2026-09-23 · [@yume_arasaki](https://x.com/yume_arasaki)

Xiaomi's V2.6-Flash: 309B MoE, 15B active, MXFP4 experts + FP8 attention straight from the factory (no post-training compression step), MIT, 1M context, omni (text/image/video/audio in). 166 GiB on disk, 65 shards. The spec that matters for Sparks: it ships quantized, so there is no quant lottery to wait out.

Serving recipe: [tonyd2wild/MiMo-V2.6-Flash-2x-DGX-Spark](https://github.com/tonyd2wild/MiMo-V2.6-Flash-2x-DGX-Spark) `@6651626`, day zero, vLLM TP2, bundled DFlash drafter k=7, fp8 KV, GMU 0.90, 300k context. Stock image needs four patched files to boot on GB10; his `setup.sh` stages them.

Strain: `mimo-v26-flash-rl/vllm-tp2/tonyd2wild/6651626`. Endpoint `:8003`. Weights pulled with HF token (166 GiB, ~2h at 17-22 MB/s), copied to both nodes over QSFP at 380 MB/s (7m25s). KV pool at 300k: **2,052,961 tokens, 6.84x** concurrency at max len.

---

## Empty context

Single stream, thinking off, T=0, unique salt. Protocol `sparkdash-*`, decode aggregate after first token.

| What I asked | C1 | C2 | C4 |
|---|---:|---:|---:|
| Count from 1 to 200 | **96.3** | 152.1 | 253.2 |
| Explain a hash map (600 tok) | **33.8** | | |
| Fifty identical Python clamps (600 tok) | **83.8** | | |
| A JSON blob of fake GPU stats | **58.0** | | |
| `2^10 + 3^5`, integer only | **29.0** | | |

Maths: it answered **1317**. The answer is 1267. exact_match 0. This is the fifth near-frontier model to flub this exact cell with thinking off (GLM wrong, DSV41 1243, Qwen 27B W4A16 1331, Qwen 27B EXL3+DFlash2 1105). The probe is suspect. The fail is still in the ledger.

Tools: `get_weather`, `finish_reason=tool_calls`, no XML in content. Clean pass.

Energy: structured C1 **344 J for 400 tokens = 0.86 J/tok**, both Sparks summed, nvidia-smi rail. Code C1 0.99 J/tok. Maths cell 11.4 J/tok (short completion, prefill-dominated).

At structured C4 (253.2 aggregate) the ceiling is **21.88M tok/day**. At US residential 18.34¢/kWh that is about **$1.13/day** in electricity for that lane.

## Concurrency (structured counting)

| Streams | Aggregate tok/s | Per stream |
|---|---:|---:|
| C1 | 96.3 | 96.3 |
| C2 | 152.1 | ~76 |
| C4 | **253.2** | ~63 |

Recipe cap is `SEQS=8`; I ran C1/C2/C4 cells. The drafter is why C1 is high: counting drafts at near-perfect acceptance. Prose drafts at ~1.2 of 7 accepted, which is why prose is 34 while counting is 96. Same hardware, different words.

## Long context

### Needle

Three codes at 5/50/95%, unique salt, protocol `needle-5-50-95-v1`.

| I aimed at | Server said | Found | Prefill |
|---:|---:|---|---:|
| 8k | 8,073 | 3/3 | 1,780 tok/s |
| 32k | 32,146 | 3/3 | 1,557 |
| 128k | 128,488 | 3/3 | 992 |
| 256k | 256,931 | **3/3** | 667 |

**12/12.** Full marks at every depth. The 300k context cap is the recipe default, not a model limit (1M native, unattempted on 2 Sparks; the pool math needs ~3.3x the per-request blocks).

### Decode after fill

256-token forced generation after the same pack family, `ctx-decode-256-v1`:

| Depth | Actual pt | Decode tok/s |
|---:|---:|---:|
| 8k | 8,033 | 26.1 |
| 32k | 32,118 | 22.1 |
| 128k | 128,467 | 18.2 |
| 256k | 256,919 | **15.2** |

Gentle falloff, about 1.7x from 8k to 256k. The hybrid SWA attention is doing its job.

## Agent turns @35k, tools live

| Client | Job | Decode | Completion | prompt_tokens | Tools |
|---|---|---:|---:|---:|---|
| OMP-shaped | tower game build | **53.3** | 2,048 tok, tool-first | 34,604 | clean, no XML |
| Hermes-shaped | research + app | **33.8** | 63 tok, tool-first | 35,832 | clean, no XML |

These are warm-prefix turns (cache hit on the packed context), shaped HTTP, not the GUIs. They are different jobs from the 600-tok generates. Printed, never subtracted.

One honest wobble from the depth grid: a tools-at-128k cell spun into repeated tool calls at 0.1 tok/s before finishing, 20.5 kJ for the turn. The storm shape this family showed in V2.5. The recipe's defaults (checkpoint temperature 1.0 / top_p 0.95 plus repetition_penalty 1.05) tamed it at 35k. Deep-context tooling is the remaining frontier.

## Community receipt, day zero

[Wësche](https://x.com/WescheNex1q/status/2102236822487134642) ran the same recipe with thinking on: 80.3K tokens in 30 minutes (44 tok/s), DFlash accepting 4-5 tokens/step in the code phase, complete file first try, zero errors. Same prompt on GLM-5.3-Flash, same pair: 79.6K tokens in 57 minutes at ~23 tok/s. His caveat: GLM's output was a bit better on polish. Faster but rougher matches my lanes.

## Recipe state (day zero)

Four patched files mount over the stock image to boot on GB10 (fused fp8 QKV load, Omni class + DFlash marker, fp8 KV that actually applies, drafter value scale). Plus a fixed dflash config (release ships a trailing comma), audio libs, and marlin MoE with DeepGEMM off (DeepGEMM #417 corrupts fp8 GEMMs on SM12x). My additions on top: the grid runner's `/tokenize` calibration 404s on this serve, so pack targets need live `usage.prompt_tokens` measurement (MiMo packs ~1.10x denser than GLM's tokenizer; the frozen GLM ratio overshoots 300k at the "128k" rung).

Numbers are one boot, one night, day-zero recipe. Treat accordingly.

---

*Isolates: `benchmarks/runs/2026-09-23_8003_*.json` in the Hamster workspace. Strain join: `mimo-v26-flash-rl/vllm-tp2/tonyd2wild/6651626`. GPU-rail is not wall power. Capex not included.*
