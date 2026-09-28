# Qwen3.8-Flash-Next hibrid48 v4.1 on two DGX Sparks (full agent grid, YaRN 1M)

2026-09-28 · [@yume_arasaki](https://x.com/yume_arasaki)

Second grid on the same pair. Same weights, same K=5, same YaRN 1M fork. Two things changed since the 26 Sep write-up.

His update: recipe `@f089598`, v4.1, FlashInfer GDN prefill. He published about +5% prefill from 8k to 256k and no decode change.

Mine: a vLLM 0.30 patch so Hermes (which omits `temperature` and sends `reasoning` instead of `reasoning_effort`) does not inherit `generation_config.json` at temperature 1.0 with thinking defaulting to `xhigh`. That was the garbled-Chinese / `function86` loop, not the 4-bit head. PR: [myllmbox/qwen38-flash-next-cluster-recipe#2](https://github.com/myllmbox/qwen38-flash-next-cluster-recipe/pull/2). Write-up: [docs/hermes-harness.md](https://github.com/yume-arasaki/qwen38-flash-next-cluster-recipe/blob/hermes-sampling/docs/hermes-harness.md).

Strain: `qwen38-hibrid48/vllm-tp2/hamster/f089598`. Endpoint `:8007`. Grid runners still send temperature 0, thinking off, so the empty-KV numbers are his v4.1 against my 26 Sep pin at `8ec444f`, not a test of the Hermes patch.

---

## Empty context

Single stream, thinking off, T=0, unique salt. Protocol `sparkdash-*`, decode aggregate after first token. Right column is the 26 Sep pin on this same pair.

| What I asked | C1 | vs 26 Sep | C2 | C4 |
|---|---:|---:|---:|---:|
| Count from 1 to 200 | **138.3** | 131.8 | 232.3 | **383.1** |
| Explain a hash map (600 tok) | **81.6** | 78.4 | 133.1 | 200.5 |
| Fifty identical Python clamps (600 tok) | **133.2** | 126.4 | 213.2 | **330.5** |
| A JSON blob of fake GPU stats | **118.6** | 111.8 | 178.0 | 290.4 |
| `2^10 + 3^5`, integer only | **57.5** | 53.8 | | |

Decode nudged up 4 to 6% on the empty-KV lanes. His card said decode would not move. I am not crowning FlashInfer for that. Run-to-run sits in that band.

Maths: exact_match 0 again. The probe is still the suspect. Fail stays in the ledger.

Tools: `get_weather`, `finish_reason=tool_calls`, no XML in content. Clean pass.

Energy: structured C1 **209 J for 400 tokens = 0.52 J/tok**, both Sparks summed, nvidia-smi rail. Code 0.59, prose 0.90, JSON 0.98 J/tok. Maths cell 7.27 J/tok.

At code C4 (330.5 aggregate) the ceiling is **28.56M tok/day**.

## Concurrency (structured counting)

| Streams | Aggregate tok/s | Wall companion |
|---|---:|---:|
| C1 | 138.3 | 133.0 |
| C2 | 232.3 | 203.3 |
| C4 | **383.1** | 352.3 |

## Long context

### Needle

Three codes at 5/50/95%, unique salt, protocol `needle-5-50-95-v1`.

| I aimed at | Server said | Found |
|---:|---:|---|
| 8k | 8,073 | 3/3 |
| 32k | 31,510 | 3/3 |
| 128k | 130,598 | 3/3 |
| 256k | 256,423 | 3/3 |
| 524k | 522,308 | 3/3 |
| 900k | 880,286 | **3/3** |

**18/18.** Same as 26 Sep.

### Decode after fill

256-token forced generation after the same pack family, `ctx-decode-256-v1`. Prefill inferred as prompt_tokens / ttft.

| Depth | Actual pt | Decode tok/s | Prefill tok/s | 26 Sep decode |
|---:|---:|---:|---:|---:|
| 8k | 7,888 | 76.8 | 3,081 | 64.1 |
| 32k | 32,067 | 65.8 | 3,216 | 66.2 |
| 128k | 130,560 | 71.4 | 3,245 | 65.4 |
| 256k | 251,634 | 71.0 | 3,113 | 64.5 |
| 524k | 512,774 | 67.0 | 2,870 | 65.7 |
| 900k | 880,245 | **70.6** | 2,577 | 64.2 |

Prefill at 128k–900k is about +6 to +7% vs 26 Sep, in line with his +5% claim. Decode at depth sits 65–77 and does not fall over. The 8k decode step (76.8 vs 64.1) is the one I would not hang a story on.

## Agent turns @35k, tools live

| Client | Job | Decode | Completion | prompt_tokens | Tools |
|---|---|---:|---|---:|---|
| OMP-shaped | tower game build | **107.4** | 2,048 tok | 35,318 | write_file, clean |
| Hermes-shaped | research + app | **81.0** | 101 tok, tool-first | 35,277 | web_search ×2, clean |

The 26 Sep OMP row printed 183.3 on a 35-token turn. That window was too thin. I flagged it suspect. This is the re-run: 2,048 tokens, 107.4 tok/s. Hermes moved 74.7 → 81.0 on a similar 100-token window.

These are still different jobs from the 600-tok generates. Printed, never subtracted.

## What the Hermes patch actually does

vLLM 0.30 applies this checkpoint's `generation_config.json` (temperature 1.0, top_p 0.95, do_sample true) when the request omits temperature. Hermes omits it. Hermes also sends `reasoning.enabled` / `effort`, which 0.30 does not map. The chat template then defaults thinking to `xhigh`. That is the loop.

Ruled out, in order: the NVFP4 output head (bf16 graft still looped), YaRN, watermarking, context past 262k.

The patch: omitted temperature becomes 0 on the chat path. A top-level `reasoning` object sets `enable_thinking`. Explicit values still win. Three table requests with temperature omitted and thinking off came back English. It does not scrub a session that already contains the bad text.

Grid numbers above are not a test of that patch. The runners already sent temperature 0.

Numbers are one boot, one night, his v4.1 plus my Hermes pins. Treat accordingly.

---

*Isolates: `benchmarks/runs/2026-09-28_8007_*.json` in the Hamster workspace. Strain join: `qwen38-hibrid48/vllm-tp2/hamster/f089598`. GPU-rail is not wall power. Capex not included. Recipe: myllmbox `@f089598`. PR: myllmbox/qwen38-flash-next-cluster-recipe#2. Power rate: EIA US residential. API ref: Grok 4.6 $6.00/M output.*
