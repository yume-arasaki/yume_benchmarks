# Qwen3.8-Flash-Next NVFP4 on two DGX Sparks

2026-09-18 · [@yume_arasaki](https://x.com/yume_arasaki)

Ran [Mia's dual-Spark recipe](https://github.com/MiaAI-Lab/Qwen3.8-Flash-Next-Dual-DGX-Sparks) (`d2f54b7`, her GitHub HEAD from 16 Sep, supersedes `c2325b2`) on two GB10 boxes. Tensor parallel 2. Weights `Mia-AiLab/Qwen3.8-Flash-Next-NVFP4`, NVFP4. YaRN stretched to 1M. MTP 3. Thinking off.

Her README prints 62.9 on count-to-200 for her own protocol line. Mine is below. Different protocol, different day. Printed, never subtracted.

The August dual numbers on this checkpoint pre-date my ledger, so they don't exist for charting. That rule exists because of me: I once published a ~27 tok/s miss on this family (wrong job, stale commit), owned it, re-ran her protocol. The correction lineage is the reason every number here carries the SHA.

I didn't mash anything into one score. Counting, writing, code, JSON, tools, and a long-context retrieve are different days at the office. Structured, prose, code, and JSON carried the concurrency axis; the rest ran one stream each.

Tok/s below is after the first token shows up. Divide by the whole wait, prompt-reading included, and you get a different, worse number. Kept apart.

Prompt sizes are what the server said it read. Power is both GPUs added together, from `nvidia-smi`, while that request was in flight. Not the wall plug. Not what I paid for the boxes.

---

## Empty context

Two runs at each concurrency, headline is the better one. Smoke check first: 17 × 19 → 323. Correct.

| What I asked | 1 stream | 2 streams | 4 streams |
|---|---:|---:|---:|
| Count from 1 to 200 | **68.2** | | |
| Explain a hash map | **50.9** | **87.8** | **141.8** |
| Fifty identical Python clamps | **66.0** | **98.1** | **202.6** |
| A JSON blob of fake GPU stats | **61.5** | **109.4** | **187.5** |
| `2^10 + 3^5`, integer only | **36.1** | | |

202.6 at four streams on code is the fastest aggregate I've printed on this desk. No sequence cap games here: the recipe holds ten, four is well inside.

The arithmetic answer is 1267. It said 1283. Speed is 36.1. Both are true. That's the fourth strain to flub this exact cell at temperature 0 (GLM, DeepSeek 4.1, a 27B, now this). At this point it's my probe, not the models. It stays in the table.

Asked it to call a weather tool the proper way, not dump XML in the reply. It did. Zero XML in content. One image turn through the vision path, thinking off: 44.2 decode on a short answer. Smoke lane, not a headline.

Count job, one stream: **452 joules** for 400 tokens on the two GPU rails, 1.13 J/tok. At US residential power (18.34¢/kWh, EIA) that's about **$1.00 a day** holding the code-C4 rate. Same firehose on Grok 4.6 output pricing ($6.00/M, frontier ref) is about **$105.03/day**. That's electricity. The boxes still cost a car.

---

## Long context

YaRN is doing the stretch to 1M on this build, so the ladder goes further than usual.

### Can it still see a needle?

Three codes, planted at 5%, 50%, and 95%. Once per depth.

| I aimed at | Server said | Found | Prefill | Time to first token |
|---:|---:|---|---:|---:|
| 8k | 7,778 | 3/3 | 2,952 tok/s | 2.6 s |
| 32k | 30,323 | 3/3 | 2,896 | 10.5 s |
| 128k | 125,854 | 3/3 | 2,505 | 50.2 s |
| 256k | 246,927 | 3/3 | 2,216 | 111.4 s |
| 524k | 522,310 | 3/3 | 1,798 | 290.4 s |
| 900k | 863,984 | 3/3 | 1,498 | 576.8 s |

**18/18.** Prefill holds 1,498 tok/s at nearly a million tokens read. Retrieval is not a speed chart.

### How fast does it talk after that fill?

Forced 256 tokens so it couldn't quit early.

| I aimed at | Server said | Decode tok/s |
|---:|---:|---:|
| 8k | 8,185 | **44.5** |
| 32k | 30,879 | **44.4** |
| 128k | 128,187 | **43.8** |
| 256k | 251,634 | **42.7** |
| 524k | 503,278 | **44.6** |
| 900k | 880,245 | **37.7** |

Flat to 524k, and 524k actually beats 8k. The one real falloff is 900k, about 15 percent off the 8k number, on a context size most rigs can't load at all. Whatever decay people expect from long context, this build trades it away gently.

### Same jobs, cache already full

| Job | ~32k | ~128k | Empty (from above) |
|---|---:|---:|---:|
| Count 1 to 200 | **58.7** | **66.7** | 68.2 |
| Hash map | **47.7** | **49.5** | 50.9 |
| Python clamps | **62.6** | **57.3** | 66.0 |
| Weather tool | called | called | called |

Depth cost little. Counting at 128k resident actually beats 32k on this run. Code gives up about ten percent from empty to 128k resident.

---

## The tool lane, where agents live

Tools with the KV already full is the cell most boxes fumble. Here it's the fastest lane on the board.

| Depth | Decode tok/s | Tool call |
|---:|---:|---|
| 32k resident | **91.7** | clean, no XML |
| 128k resident | **89.7** | clean, no XML |

Near-flat tool decode at a 4x depth range. This is the number I care about for agent work, not the count-to-200.

---

## Warm session, like an agent already in the middle of work

About 35k already in the prompt. Tools attached. One turn each.

| I asked | Prompt tokens | What it did | tok/s |
|---|---:|---|---:|
| Write a tower-stacking game as one HTML file | 34,050 | 75 tokens of plan, then a clean bash call | **56.4** |
| Look up spaced repetition, then build a flashcard app | 34,643 | 100 tokens of plan, then two web searches | **36.8** |

One wrote a game plan, one reached for search. Different jobs, not charted against each other. Tower run burned 1,475 joules, 19.7 per token. Research turn 1,624 joules, 16.2 per token. Both tool-first, both streamed clean.

---

## What I didn't run

No thinking-on lanes. No wall-plug power. No concurrency at depth yet — four streams on an empty KV is not four streams at 128k resident, and I won't pretend it is. That's the next run. No TP=1 comparison: one box on this checkpoint is a different strain, and charting them together is how I got burned the first time.

---

## Credit

Recipe: [MiaAI-Lab/Qwen3.8-Flash-Next-Dual-DGX-Sparks](https://github.com/MiaAI-Lab/Qwen3.8-Flash-Next-Dual-DGX-Sparks) `@d2f54b7`.
Quant: [Mia-AiLab/Qwen3.8-Flash-Next-NVFP4](https://huggingface.co/Mia-AiLab/Qwen3.8-Flash-Next-NVFP4).
Power rate: EIA, US residential. API prices from 18 Sep 2026 (Grok 4.6 first-party list, $6.00/M out).
Correction lineage: [my own TP=1 miss and re-run](https://x.com/yume_arasaki/status/2096835295706747155).
