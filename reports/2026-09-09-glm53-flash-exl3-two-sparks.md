# GLM-5.3-Flash EXL3 on two DGX Sparks

Field report. 2026-09-09. [@yume_arasaki](https://x.com/yume_arasaki)

We ran [MiaAI-Lab's dual-Spark EXL3 recipe](https://github.com/MiaAI-Lab/GLM-5.3-Flash-EXL3-2x-DGX-Sparks) (`9c0794b`) on two GB10 boxes, tensor parallel 2. Weights: `Mia-AiLab/GLM-5.3-Flash-EXL3-TR3-4bpw`. Drafter on. Thinking off. Throughput below is **decode after the first token**, not tokens divided by the whole wait including prefill.

Her README structured bar on this recipe family is **62.9** tok/s. Ours is **66.4**. Same job class (count 1 to 200). Different pin and different desk. We are not printing a percent delta.

---

## Short context

| | |
|---|---|
| Hardware | 2× DGX Spark (GB10), TP=2 |
| Model | GLM-5.3-Flash EXL3, 4bpw |
| Thinking | off |
| Speculative decode | on (k=7) |
| What “tok/s” means here | completion tokens / time after first content token |

Power numbers are **GPU draw on both cards, added together**. Not wall power. Not the price of the Sparks.

---

## Empty context

Two repeats at each concurrency for structured. Peak of the two is the headline.

| Job | C1 | C2 | C4 |
|---|---:|---:|---:|
| Count 1→200 | **66.4** | **133.4** | **187.6** |
| Hash-map essay | **28.5** | — | — |
| Repetitive Python (`clamp_00`…`clamp_49`) | **61.3** | — | — |
| JSON object | **44.7** | — | — |
| Short arithmetic | **29.9** | — | — |

Arithmetic asked for `2^10 + 3^5` as a bare integer (**1267**). Decode is 29.9. The integer itself did not match. We still report the speed.

Tool call (weather, structured `tool_calls`, no XML dumped into the text): **pass**.

Count-1→200 C1 energy on the GPU rails: **417 J** for 400 tokens (~1.0 J/tok). At US residential 18.34¢/kWh, that C1 rate is about **$0.31/day** of GPU electricity if you could hold it. Same token rate on Grok 4.6 output pricing ($6/M) is about **$34/day**. Same-model API list price for GLM-5.3-Flash ($0.50/M out) is about **$2.87/day**. The boxes are not free. The column is electricity, not capex.

---

## Long context

Packed filler, then the task. Prompt size is what the server counted, not the size we aimed at (the pack runs long on this tokenizer).

### Did it find the needles?

Three unique codes planted at 5%, 50%, 95%. One pass per depth.

| Aimed | Server prompt tokens | Found | Prefill tok/s | Time to first token |
|---:|---:|---|---:|---:|
| ~8k | 17 968 | 3/3 | 1 483 | 12 s |
| ~32k | 69 454 | 3/3 | 1 615 | 43 s |
| ~128k | 279 170 | 3/3 | 1 549 | 180 s |
| ~256k | 557 116 | 3/3 | 1 458 | 382 s |

**12/12.** That is retrieve, not a speed curve.

### Decode after fill (256 tokens)

This is the speed-vs-depth line. Not the empty-context 66.4.

| Aimed | Server prompt tokens | Decode tok/s |
|---:|---:|---:|
| ~8k | 17 208 | **21.6** |
| ~32k | 67 967 | **21.8** |
| ~128k | 284 962 | **24.3** |
| ~256k | 557 086 | **24.7** |

No fall-off on this box. Slightly up.

### Same jobs with the cache already full

| Job | ~32k | ~128k | Empty C1 (from above) |
|---|---:|---:|---:|
| Count 1→200 | **65.7** | **65.3** | 66.4 |
| Hash-map essay | **28.2** | **30.2** | 28.5 |
| Repetitive Python | **61.6** | **63.6** | 61.3 |
| Tool call | called (12 tok) | called (12 tok) | pass |

The long jobs hold. The tool rows are a handful of tokens. Do not read 0.3 tok/s as “tools are slow.”

---

## Agent-shaped turns (~35k already in the prompt)

Not the OMP app. Not Hermes the product. One HTTP turn, tools attached, thinking off, session already warm.

| Turn | Prompt tokens | What happened | Decode tok/s |
|---|---:|---|---:|
| “Build a browser tower-stacking game, one HTML file” | 34 080 | Wrote ~2048 tokens, then a file-write tool call | **43.8** |
| “Research local SRS, then build a flashcard app” | 35 298 | 55 tokens, then two web searches | 22.5 on 55 tok |

Those are not the same job. The second turn went to tools first. We are not ranking 22.5 against 43.8.

---

## What this is not

- Not a single “the model is X tok/s.”
- Not a beat on Mia’s 62.9. Different pin, different desk.
- Not wall power, RAPL, or the cost of the hardware.
- Not a recipe dump. Serve stack is hers; we credit the repo and the SHA.
- Not 15/15 needles. One pass per depth is 3/3, four depths 12/12.

---

## Credit

Recipe and image: [MiaAI-Lab/GLM-5.3-Flash-EXL3-2x-DGX-Sparks](https://github.com/MiaAI-Lab/GLM-5.3-Flash-EXL3-2x-DGX-Sparks) `@9c0794b`.  
Electricity rate: EIA US residential, June 2026, 18.34¢/kWh.  
Grok 4.6 and Z.AI GLM-5.3-Flash list prices as of 2026-09-09.
