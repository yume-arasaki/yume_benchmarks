# GLM-5.3-Flash TensorFold on two DGX Sparks (engine swap, full grid)

2026-10-01 · [@yume_arasaki](https://x.com/yume_arasaki)

Engine swap week. Same pair of Sparks, same 200 Gb/s cable, a different thing running the model. Last grid here was [Mia's EXL3 recipe on vLLM](reports/2026-09-09-glm53-flash-exl3-two-sparks.md), 9 September. This one is [jayleaton's TensorFold recipe](https://github.com/jayleaton/glm53-tensorfold-spark) `@50641cf`, serving an abliterated EXL3 build based on Orca Router's abliterated model. Full power AI.

TensorFold is new, so this write-up runs longer than usual. The short version: same rig, same protocols, same quant class as my 9 Sep grid, and decode moved 1.57x to 2.03x. The engine did that.

Strain: `glm53-flash-uncensored-exl3/tensorfold-tp2/hamster/50641cf`. Endpoint `:8008`, two-rank TP=2, thinking off for the grid lanes, T=0. Isolates in `benchmarks/runs/`, nights of 30 Sep and 1 Oct.

---

## What TensorFold is, and why I cared

Decode on these boxes is memory-bound. 273 GB/s of bandwidth, and every token means reading every active weight. While that read crawls past, the compute units mostly idle. Speculative decoding fills the gap: a draft head proposes several tokens, the model verifies them in one pass. A verified token is one the model itself would have picked, so output matches serial decoding. The tax is zero versus the same weights decoded normally.

Every engine drafts now. The llama.cpp control test is what convinced me the verifier is where the money is: same drafter, same weights, llama-server gets maybe 1.5x on code and a slowdown on prose. TensorFold's tree verification in a single pass is the best I have measured on this hardware.

The recipe pins TensorFold as an unmodified submodule and applies 76 patches at image build: a latent (absorbed MLA) KV cache in FP8 holding a shared 1M-token pool, batching up to 4 requests, image input through the checkpoint's own vision tower, structured output through xgrammar, a RoCE all-gather, deeper drafting and verify windows. Every patch is off by default and the configs turn on the measured set. The repo's own gates: drafted equals serial 10 of 10, twice. Batched equals solo, 92 of 92. MMLU-200 88.0 percent.

## Deployment, as it happened

My agent staged the whole thing at night. Pipeline start 21:35, weights verified 21:59, image built 22:10, rank 1 shipped to the second Spark 22:18. First serve boot took 604 seconds because it compiles kernels; the restart after a slot-count fix was ready in 33 seconds. Canary came back at 5.71 tokens per draft round, in line with the recipe's acceptance tables.

Since then it has served 285 requests and 5.3M prompt tokens with zero errors, per `/health`. The model was live while I wrote this.

## Empty context

Two runs a lane, headline is the better one. Decode aggregate after first token. Right column is my 9 Sep vLLM EXL3 grid on this same pair, same protocols, same port.

| What I asked | 1 stream | vs 9 Sep | 2 streams | 4 streams |
|---|---:|---:|---:|---:|
| Count from 1 to 200 | **109.1** | 66.4 | 137.3 | 134.4 |
| Explain a hash map (600 tok) | **57.7** | 28.5 | 71.3 | 89.4 |
| Fifty identical Python clamps | **96.5** | 61.3 | 137.3 | 134.4 |
| A JSON blob of fake GPU stats | **80.2** | 44.7 | 108.6 | **146.5** |
| `2^10 + 3^5`, thinking off | 23.1 | | | |

Count to count, 1.64x. Prose, 2.03x. Clamps, 1.57x. JSON, 1.79x. Same quant class both sides, EXL3 4-bit. One caveat stays on the table: the 9 Sep side is Mia's TR3 pack, this side is the abliterated build. Not identical weights. This is as close to a sterile engine A/B as this desk produces, and I am printing deltas with that label attached.

C2 and C4 do not scale the way vLLM's did. That is real and it is below.

The maths probe failed again, thinking off, exact_match 0. With thinking on, exact_match 1. That cell has now fooled GLM, DSV41, two 27B quants, MiMo and hibrid48. It is the probe, not the models. Ledger stays probe-suspect.

Tools: `get_weather`, `tool_calls` present, no XML dumped in content. Clean pass both nights.

## Concurrency, and where it stops

Counting sweep, aggregate decode:

| Streams | Aggregate tok/s |
|---:|---:|
| 1 | 49.6 |
| 2 | 65.3 |
| 4 | **86.3** |
| 8 | 55.5 |

Four is the ceiling. `GLM53_TF_BATCH=4` is a config value, not a discovery, but the collapse past it is worth printing: at 8 streams the box is a queue, aggregate falls back below the two-stream number. vLLM on this rig serves 16 seats. TensorFold serves 4 exact ones. If you need a bus, this is not your engine. If you need one fast seat for an agent, it is.

## Draft acceptance, the agent number

The recipe's W11 study measures per-position draft acceptance by content class, DFlash2 drafter at depth 7. Conditional acceptance on agent-like content (file rewrites, unified diffs, JSON records, shell plans) sits at 0.90 across all seven positions. That is 5.76 tokens committed per verify round. Prose drafts at 2.69. The engine is literally fastest on the work agents do all day, which is the whole reason it is on my rack.

Canary on my serve: 5.71 tokens per round. Matches.

## Long context

Native 1M window. Ladder forced to 256 tokens so it could not quit early:

| I aimed at | Server said | Decode tok/s |
|---:|---:|---:|
| 8k | 8,074 | **46.4** |
| 32k | 32,518 | **48.4** |
| 128k | 129,142 | **44.1** |
| 256k | 257,680 | **45.0** |
| 524k | 513,847 | **40.3** |
| 900k | 880,245 | **40.1** |

Flat. 13 percent down at 900k from the 8k lane. For contrast, the 9 Sep vLLM grid read 21.6 to 24.7 across a ladder half this length. The latent KV cache is doing what it claims.

Needle, three codes at 5, 50 and 95 percent, unique salt: **18/18**, deepest at 900k. Prefill across the ladder ran 1,497 down to 1,320 tok/s. vLLM cold prefill on this rig was faster, about 1,800. That trade is below.

Same jobs with KV already full: count 65.3 at 128k context against 109.1 empty. Job-at-depth rows carry one 12-token turn that read 32,593.9 tok/s, a two-point timing artifact. Excluded, recorded here so nobody finds it in the JSON and thinks I hid it.

## Warm sessions, like an agent mid-work

About 35k in the prompt, tools attached, one turn each. Not the real GUIs.

| I asked | Prompt tokens | What it did | tok/s |
|---:|---|---|---:|
| Write a tower-stacking game as one HTML file | 35,091 | Wrote a full 2,048 tokens, then tried to save a file | **68.3** |
| Look up spaced repetition, then build a flashcard app | 36,290 | 65 tokens of intent, then two web searches | 46.6 |

Same shapes as 9 Sep (43.8 then), both faster now. The second row is a 65-token turn. Do not read it as a speed row.

## The recipe's own RigMark, printed separately

The author's receipts, his NVFP4 baseline, his daily driver before this engine. Not my numbers, not my grid. Code decode 42.6 to 68.6, prose 22.2 to 43.2, structured 54.6 to 88.2, three-run averages. Under load 1.74x at one stream down to 1.34x at four. Cold prefill 1,598 against 1,835 tok/s, the honest loss. Warm replay 0.29x, because vLLM's prefix replay is very good. First-token under four streams 1.9 seconds against 0.78.

## Energy and cost

GPU rail, both Sparks summed, nvidia-smi. Structured C1 reads 1.27 J/tok, code 1.37, prose 2.24, JSON 2.72. At the fastest concurrency the recipe holds, JSON C4 at 146.5 aggregate, that is 12.66M tokens a day ceiling, about $0.88 a day of electricity at EIA residential rates. The same tokens through Claude Opus 5.5 at $20 per million output is $253 a day. Printed, never claimed as savings. Duty 1.0 is a lie nobody runs.

## What it cost me

Cold prefill, 13 to 15 percent down. Warm replay, badly behind vLLM. Four seats, hard. C4 first-token under load 2.4x worse. If your work is short prompts and huge shared prefixes, keep vLLM. This engine is a bet on decode, and on exactness as a feature.

## What I am watching

Prefill. Upstream release notes show cold prefill at 128k up 37 percent in the newest release alone. If replay closes too, the last reason to keep vLLM on these boxes goes away.

The stacking question. My Qwen 3.8 Flash strain gets speed from cheaper weights. TensorFold gets it from richer weight reads. Nobody has tried both at once.

The Mac lane. Same engine, MLX path, same drafted-equals-serial gates. A Mac mini has 273 GB/s and the same disease.

---

## Credit

Engine: [ashhart/TensorFold](https://github.com/ashhart/TensorFold), MIT, Ash Hart. Recipe: [jayleaton/glm53-tensorfold-spark](https://github.com/jayleaton/glm53-tensorfold-spark) `@50641cf`. Weights: [neko-legends/GLM-5.3-Flash-Uncensored-EXL3](https://huggingface.co/neko-legends/GLM-5.3-Flash-Uncensored-EXL3), abliterated build based on Orca Router's abliterated model, upstream GLM-5.3-Flash by Zhipu. Opus 5.5 pricing from platform.claude.com, 1 Oct 2026. Power rate: EIA, US residential, June 2026. The numbers are mine.
