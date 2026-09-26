# Qwen3.8-Flash-Next hibrid48 on two DGX Sparks (full agent grid, YaRN 1M)

2026-09-26 · [@yume_arasaki](https://x.com/yume_arasaki)

The model everyone's recipes target, on the fastest community quant of it yet. hibrid48 is vr8vr8's build: hibrid47 with exactly one tensor changed, the output head requantized from bf16 to NVFP4. That head is 1.18 GiB and speculative decoding reads it about 5.4 times per step (four drafter proposals plus the verify pass), which put it at 27% of every decode step. At 4-bit it is 0.33 GiB. Same body, same drafter, engine steps 16.9 → 22.0/s on his card. On a bandwidth-starved box, that is the whole ballgame.

Serving recipe: [bilikaz/qwen38-flash-next-cluster-recipe](https://github.com/bilikaz/qwen38-flash-next-cluster-recipe) `@8ec444f` (v6 image). Weights: [myllmbox/Qwen3.8-Flash-Next-hibrid48](https://huggingface.co/myllmbox/Qwen3.8-Flash-Next-hibrid48). vr8vr8 is bilikaz on GitHub; myllmbox builds the weights. One person's ecosystem, three names.

Strain: `qwen38-hibrid48/vllm-tp2/hamster/8ec444f-yarn`. My fork adds one thing: YaRN factor 4.0, context to 1M (his default tops at 262k). Static YaRN changes short context too, so the stock build and mine are separate strains. His stock short isolates on this same rig: structured 107.5, code 127.2.

First boot was rough: garbled text mixed with Chinese characters, thinking on and off both, until hot patches. Clean after. Also on record from his card: stock vLLM 0.29 shape-mismatches this checkpoint (the quantized head needs a two-line `quant_config` patch, shipped in his image).

---

## Empty context

Single stream, thinking off, T=0, unique salt. Protocol `sparkdash-*`, decode aggregate after first token.

| What I asked | C1 | C2 | C4 |
|---|---:|---:|---:|
| Count from 1 to 200 | **131.8** | 223.1 | **369.2** |
| Explain a hash map (600 tok) | **78.4** | 131.8 | 198.4 |
| Fifty identical Python clamps (600 tok) | **126.4** | 204.1 | **321.6** |
| A JSON blob of fake GPU stats | **111.8** | 160.5 | 266.7 |
| `2^10 + 3^5`, integer only | **53.8** | | |

Maths: exact_match 0. Sixth near-frontier strain to flub this exact cell with thinking off (GLM, DSV41 1243, Qwen 27B W4A16 1331, Qwen 27B EXL3 1105, MiMo 1317 before it). The probe is suspect. The fail is still in the ledger.

Tools: `get_weather`, `finish_reason=tool_calls`, no XML in content, 27 tokens in 0.48s. Clean pass.

Energy: structured C1 **219 J for 400 tokens = 0.55 J/tok**, both Sparks summed, nvidia-smi rail. Code 0.61, prose 0.90, JSON 0.98 J/tok. Maths cell 8.06 J/tok (short completion, prefill-dominated).

At code C4 (321.6 aggregate) the ceiling is **27.79M tok/day**. At US residential 18.34¢/kWh that is about **$0.38/day** in electricity for that lane.

## Concurrency (structured counting)

| Streams | Aggregate tok/s | Wall companion |
|---|---:|---:|
| C1 | 131.8 | 126.7 |
| C2 | 223.1 | 196.5 |
| C4 | **369.2** | 340.5 |

Sum-of-streams with the wall beside it, as always. Counting drafts at near-perfect acceptance and K=5 banks up to six tokens per step, which is why this lane runs away from prose.

## Long context

### Needle

Three codes at 5/50/95%, unique salt, protocol `needle-5-50-95-v1`.

| I aimed at | Server said | Found |
|---:|---:|---|
| 8k | 8,223 | 3/3 |
| 32k | 31,512 | 3/3 |
| 128k | 128,226 | 3/3 |
| 256k | 237,433 | 3/3 |
| 524k | 503,318 | 3/3 |
| 900k | 880,284 | **3/3** |

**18/18.** Full marks at every depth, 900k included.

### Decode after fill

256-token forced generation after the same pack family, `ctx-decode-256-v1`:

| Depth | Actual pt | Decode tok/s |
|---:|---:|---:|
| 8k | 8,036 | 64.1 |
| 32k | 32,660 | 66.2 |
| 128k | 130,560 | 65.4 |
| 256k | 251,634 | 64.5 |
| 524k | 493,783 | 65.7 |
| 900k | 880,244 | **64.2** |

Flat. 64 → 64 across a 112x context range. Prefill slid from 2,787 to 2,396 tok/s across the ladder and decode did not care.

## Agent turns @35k, tools live

| Client | Job | Decode | Completion | prompt_tokens | Tools |
|---|---|---:|---|---:|---|
| OMP-shaped | tower game build | **183.3** | 35 tok, tool-first | 34,685 | clean, no XML |
| Hermes-shaped | research + app | **74.7** | 98 tok, tool-first | 36,545 | clean, no XML |

Honesty note on the OMP row: 35 tokens timed over a ~0.2s decode window (wall 11.5s, TTFT 11.31s). That implies ~29.6 engine steps/s when the build measures 22.0. Two-point artifact, single rep, same failure family I have caught before on this rig. Printed, flagged suspect, generate re-run queued. The Hermes row (98 tokens) is the honest agent number.

## Same rig, separate strain

Mia's dual-Spark NVFP4 recipe `d2f54b7`, benched by me on this same pair with the identical grid two weeks prior: structured 68.2, prose 50.9, code 66.0, JSON 61.5, needle 18/18, OMP 56.4, research 36.8. Different weights (bf16 head), MTP K=3, fp8 KV. Not a like-for-like A/B and I do not chart them as one model. As same-instrument context: every clean lane reads 1.5x to 2.0x here.

## Why bf16 KV

fp8 KV boots and fits more tokens, but his own 5+5 A/B measured it costing 0.3 accepted draft tokens per step (4.23 → 3.92). Acceptance is speed with speculative decoding. The bigger cache format buys throughput; it is not memory spent for its own sake.

## Community receipts

- [Ahmad Osman](https://x.com/TheAhmadOsman/status/2103250363339952213) ran Mac Studio M5 Ultra vs 2x DGX Spark on DeepSeek V4 Flash: Sparks 2.41x faster on prefill, M5 Ultra 6.4% ahead on generation, first token 1.3s vs 2.9s. His caveat: the Mac still lacks optimized kernels. Different model than this report, treat as directional.
- A hands-on [M5 Ultra Mac Studio review](https://www.youtube.com/watch?v=cyUDevB5PxA) (256GB, 36-core CPU / 80-core GPU) puts it ~46% faster than the M3 Ultra in LM Studio on Qwen 3.5 27B 8-bit MLX, Lux 2 at 2048x2048 at 6s vs 16s, Geekbench 52,133 multi, Cinebench GPU 140,369. Creator-workload numbers, not agent-lane, but the same shape: the Ultra wins single-stream comfort, the Spark cluster wins prefill, throughput and the tuned recipe ecosystem. Nobody ships an M5 Ultra cluster recipe at this level yet.

Numbers are one boot, one night, community recipe pinned one commit. Treat accordingly.

---

*Isolates: `benchmarks/runs/2026-09-26_8007_*.json` in the Hamster workspace. Strain join: `qwen38-hibrid48/vllm-tp2/hamster/8ec444f-yarn`. GPU-rail is not wall power. Capex not included. Recipe and weights: bilikaz / myllmbox. Power rate: EIA US residential, 2026-09. API ref: Grok 4.6 $6.00/M output.*
