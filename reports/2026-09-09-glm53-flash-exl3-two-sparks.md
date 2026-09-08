# GLM-5.3-Flash EXL3 on two DGX Sparks

2026-09-09 · [@yume_arasaki](https://x.com/yume_arasaki)

[MiaAI-Lab dual-Spark EXL3](https://github.com/MiaAI-Lab/GLM-5.3-Flash-EXL3-2x-DGX-Sparks) `9c0794b` on two GB10 boxes, TP=2. Weights `Mia-AiLab/GLM-5.3-Flash-EXL3-TR3-4bpw`. Drafter on (k=7). Thinking off. Pin taken at the start of the run.

Her public structured bar on this family is **62.9** tok/s. Ours is **66.4**. Same job class. Different pin, different desk. No delta.

These are **selected cells**, not a blended score, not the full matrix.

---

## How to read this

Tok/s is completion tokens over time **after the first content token**. Wall (including prefill) is a different clock. We do not mix them.

A count job is not an essay. An essay is not a tool call. A retrieve hit is not a speed curve. A 55-token search turn is not a 2048-token generate. If two rows would need a footnote to sit in the same table, they do not sit in the same table.

Prompt length is whatever the server counted. We aimed at round depths. The pack runs long on this tokenizer. The column is the counted size.

Power is GPU draw on **both** cards, added. Not wall. Not RAPL. Not the price of the Sparks. Joules ride the same request as the tokens.

We freeze the job. If the timing changes, it is a new row.

---

## Kit

| | |
|---|---|
| Hardware | 2× DGX Spark (GB10), TP=2 |
| Model | GLM-5.3-Flash EXL3, 4bpw |
| Thinking | off |
| Speculative decode | on, k=7 |

---

## Empty context

Structured: two repeats per concurrency. Headline is the peak. Other jobs: C1 only. That is a choice.

| Job | 1 stream | 2 streams | 4 streams |
|---|---:|---:|---:|
| Count 1→200 | **66.4** | **133.4** | **187.6** |
| Hash-map essay | **28.5** | | |
| Repetitive Python | **61.3** | | |
| JSON object | **44.7** | | |
| Short arithmetic | **29.9** | | |

Arithmetic wanted a bare integer (`2¹⁰ + 3⁵` = 1267). Speed 29.9. The integer did not match. Both facts stay.

Structured tool call (no XML leaked into the text): **pass**.

Count C1, GPU rails: **417 J** / 400 tokens. US residential 18.34¢/kWh (EIA, June 2026). If you could hold that C1 rate: about **$0.31/day** local electricity vs about **$34/day** Grok 4.6 output ($6/M) vs about **$2.87/day** GLM-5.3-Flash list ($0.50/M out). Electricity, not capex. A second, lower duty exists. It is not this column.

---

## Depth

Packed filler, then the task. Three clocks. They are not substitutes.

### Retrieve

Three unique codes at 5 / 50 / 95. One pass per depth. Not five repeats of 32k.

| Aimed | Counted tokens | Hits | Prefill tok/s | Time to first token |
|---:|---:|---|---:|---:|
| 8k | 17 968 | 3/3 | 1 483 | 12 s |
| 32k | 69 454 | 3/3 | 1 615 | 43 s |
| 128k | 279 170 | 3/3 | 1 549 | 180 s |
| 256k | 557 116 | 3/3 | 1 458 | 382 s |

**12/12.** Retrieve only.

### Decode after fill

256 tokens, forced length. This is the speed-vs-depth line. Not the empty-context 66.4.

| Aimed | Counted tokens | Decode tok/s |
|---:|---:|---:|
| 8k | 17 208 | **21.6** |
| 32k | 67 967 | **21.8** |
| 128k | 284 962 | **24.3** |
| 256k | 557 086 | **24.7** |

No fall-off. Slightly up.

### Job with the cache already warm

| Job | ~32k | ~128k | Empty (from above) |
|---|---:|---:|---:|
| Count 1→200 | **65.7** | **65.3** | 66.4 |
| Hash-map essay | **28.2** | **30.2** | 28.5 |
| Repetitive Python | **61.6** | **63.6** | 61.3 |
| Tool call | called | called | pass |

The long jobs hold. Tool rows are a handful of tokens. That is not a speed.

---

## Warm session, agent-shaped

One HTTP turn. Tools on. Thinking off. About 35k already in the prompt. Not the GUI.

| Ask | Counted tokens | What it did | Decode tok/s |
|---|---:|---|---:|
| Build a tower-stacking game, one HTML file | 34 080 | ~2048 tokens, then a write | **43.8** |
| Research local spaced repetition, then build the app | 35 298 | 55 tokens, then two searches | — |

The second turn went to tools. We do not rank it against 43.8.

---

## Left on the table

Concurrency sweep only on the count job. Arithmetic at depth, JSON at depth, a dedicated skip-tax cell, thinking-on, and the GUIs: not this write-up.

Wall power: not measured on this kit the way GPU-rail is.

---

## Credit

Serve recipe and image: [MiaAI-Lab/GLM-5.3-Flash-EXL3-2x-DGX-Sparks](https://github.com/MiaAI-Lab/GLM-5.3-Flash-EXL3-2x-DGX-Sparks) `@9c0794b`.  
Electricity: EIA US residential, June 2026. API list prices as of 2026-09-09.
