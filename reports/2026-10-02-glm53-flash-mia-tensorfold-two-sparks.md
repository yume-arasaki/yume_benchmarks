# GLM-5.3-Flash on Mia's own TensorFold recipe, two DGX Sparks (aligned weights, flat falloff)

2026-10-02 · [@yume_arasaki](https://x.com/yume_arasaki)

Yesterday's grid was [jayleaton's TensorFold recipe](reports/2026-10-01-glm53-flash-tensorfold-two-sparks.md) serving an abliterated build. Interesting engine, wrong weights for a clean read. Same night, [Mia shipped their own TensorFold kit](https://mia-ai.net/models/GLM-5.3-Flash-EXL3-2x-DGX-Sparks-TensorFold) — official recipe, aligned weights, same quant class as my [9 Sep vLLM grid](reports/2026-09-09-glm53-flash-exl3-two-sparks.md). So the desk ran it: full agent grid, same frozen clocks, same pair of Sparks, same port.

Strain: `glm53-flash-exl3/tensorfold-tp2/mia/1f3d909`. TensorFold v0.5.0 plus 52 patches, DFlash2 and copy drafts, FP8 KV pool at 2.92M tokens, dense q4, RoCE one-shot gathers, vision on. Weights `Mia-AiLab/GLM-5.3-Flash-EXL3-TR3-4bpw` at the pinned revision. Thinking off for the grid lanes, T=0, unique salt per stream.

---

## Deployment

Boring, which is the compliment. One `prepare.sh` pulled the pinned image by digest on both nodes and resolved the checkpoint — our cache already held blobs matching the pin's hashes, so the "176 GB download" was hardlinks. First boot took five minutes because it compiles seven GB10 CUDA extensions, cached after. Smoke through both ranks, then live. The swap was one stop and one start; the old serve had held the port for a day.

## Their numbers, my numbers

Mia posts 1-stream 60.4 tok/s prose, 114.7 structured, four concurrent at 108.8 combined. My grid: prose C1 **59.2**, structured C1 **112.8**, prose C4 **104.8** peak. Dead on, within run-to-run noise. The concurrency rows went past their table: structured C2 **175.8** and C4 **249.8** against their 147.6 / 227.9. I don't know if that's their conservative clock cap or my salt scheme. Printing both, mine next to theirs.

## The grid

Two runs a lane, headline is the better one. Decode aggregate after first token.

| What I asked | 1 stream | 2 streams | 4 streams |
|---|---:|---:|---:|
| Count from 1 to 200 | **112.8** | 175.8 | **249.8** |
| Explain a hash map (600 tok) | **59.2** | 83.8 | **104.8** |
| Fifty identical Python clamps | **95.4** | 153.1 | **231.3** |
| A JSON blob of fake GPU stats | **76.4** | 119.1 | **188.7** |

## The part that matters: depth

This is why I keep grids on this engine. Ctx-decode, 256 forced tokens after packed filler:

| Depth | Decode tok/s |
|---|---:|
| 8k | 54.9 |
| 32k | 46.2 |
| 131k | 54.2 |
| 262k | 50.8 |
| 524k | 49.9 |
| 874k | 44.6 |

Flat. At 874k actual tokens — eighty-three percent of the full window — decode still holds 81 percent of the empty-context rate. My vLLM strains fall off a shelf here — the hibrid48 grid went 26.1 to 15.2 across a shorter ladder. The latent FP8 cache holds decode at depth. That is the whole ballgame for long agent sessions, and it is not a small-model trick: this is a 309B-class MoE.

Needle at six depths, 5/50/95 positions: **18 of 18** found, including the 900k row (879k actual tokens). Prefill at depth: 1,888 tok/s at 32k, 1,340 at 524k — close to their posted 1,979.

Jobs keep their shape at 131k: structured 107.4, prose 59.8, code 94.0, tools 212.4. Client towers at 35k: the OMP tower writes 2,048 tokens at **78.1**, tool calls clean. The Hermes client lands a 65-token tool-call turn at **51.8**. Tool parsing is exact — a `get_weather` call with `{"city":"Tokyo"}`, finish reason `tool_calls`, no XML drool.

The generate follow-up — turn two, no tools, 2,048 streamed tokens of real HTML after the tool round-trip — lands **69.5** on the OMP tower and **69.7** on the Hermes client, both at ~32k context. That is the number an agent actually feels building something, and it sits above the empty-context prose lane because the code-shaped drafting carries it.

The concurrency ladder tops at four streams on this serve by design. Prose-600 aggregates: 1 stream 53.5, 2 streams 72.8, 4 streams **97.3**. Past four the requests queue — eight concurrent still completes (67.5 aggregate, 11.8s median first-token) but the chart ceiling is four. Energy per token bottoms at C4: 1.13 J/tok on the summed rail.

## What still fails

Arithmetic. `2^10 + 3^5` comes back wrong with thinking off and with thinking on. Same wobble as yesterday's strain and the MiMo grid before it. The drafting path is exact — drafted equals serial, verified per round — so this is the model, not the engine. It gets charted as a fail, not footnoted.

## Energy

GPU rail, both Sparks summed. Structured C1 reads about 1.27 J/tok, code 1.36, prose 2.26. At the C4 structured ceiling of 249.8 aggregate that is roughly 21.6M tokens a day, about $1.50 a day of electricity at EIA residential rates. The same tokens through Claude Opus 5.5 at $20 per million output is $432. Printed, never claimed as savings.

## Three serves, one port, what I know now

Nine days, three full grids on GLM-5.3-Flash at this desk: Mia's vLLM EXL3 (9 Sep), jayleaton's TensorFold on abliterated weights (1 Oct), Mia's TensorFold on aligned weights (tonight). The engine moved decode 1.5 to 2x. The aligned kit moves nothing versus the abliterated build on speed — parity or a point or two — which means yesterday's speed story was never about the uncensored weights. It was always the engine. And the depth falloff, the thing that actually kills agent sessions, only TensorFold has fixed so far.

## Credit

Engine: [ashhart/TensorFold](https://github.com/ashhart/TensorFold), MIT, Ash Hart. Recipe: [MiaAI-Lab/GLM-5.3-Flash-EXL3-2x-DGX-Sparks-TensorFold](https://github.com/MiaAI-Lab/GLM-5.3-Flash-EXL3-2x-DGX-Sparks-TensorFold) `@1f3d909`, recipe page [mia-ai.net](https://mia-ai.net/models/GLM-5.3-Flash-EXL3-2x-DGX-Sparks-TensorFold). Weights: [Mia-AiLab/GLM-5.3-Flash-EXL3-TR3-4bpw](https://huggingface.co/Mia-AiLab/GLM-5.3-Flash-EXL3-TR3-4bpw), a byte-identical mirror of brandonmusic's TR3 pack, upstream GLM-5.3-Flash by Zhipu. Opus 5.5 pricing from platform.claude.com, 2 Oct 2026. Power rate: EIA, US residential, June 2026. The numbers are mine.
