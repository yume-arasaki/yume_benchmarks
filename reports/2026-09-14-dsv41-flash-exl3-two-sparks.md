# DeepSeek V4.1 Flash EXL3 on two DGX Sparks

2026-09-14 · [@yume_arasaki](https://x.com/yume_arasaki)

Ran [Mia's dual-Spark EXL3 recipe](https://github.com/MiaAI-Lab/DeepSeek-v4.1-Flash-EXL3-2x-DGX-Sparks) (`8530568`) on two GB10 boxes. Tensor parallel 2. Weights `Mia-AiLab/DeepSeek-V4.1-Flash-EXL3-2.9bpw`, 2.9 bits average. DSpark draft on, k=3. Thinking off.

Her repo doesn't post a count-to-200 number for this build, so there's no reference to print. Just mine.

The recipe ships with a two-sequence cap (`MAX_NUM_SEQS=2`). That means four "concurrent" streams are really two pairs taking turns. I benched it as it shipped and I say so every time a 4-stream number appears.

I didn't mash anything into one score. Counting, writing, code, JSON, tools, and a long-context retrieve are different days at the office. The shape lanes ran one stream each. Structured carried the concurrency axis.

Tok/s below is after the first token shows up. Divide by the whole wait, prompt-reading included, and you get a different, worse number. Kept apart.

Prompt sizes are what the server said it read. I aimed at 8k / 32k / 128k / 256k. This tokenizer overshoots about 2.1x. I report the counted size.

Power is both GPUs added together, from `nvidia-smi`, while that request was in flight. Not the wall plug. Not what I paid for the boxes.

---

## Empty context

Two runs at each concurrency, headline is the better one. Smoke check first: 17 × 19 → 323. Correct.

| What I asked | 1 stream | 2 streams | 4 streams |
|---|---:|---:|---:|
| Count from 1 to 200 | **45.5** | **74.6** | **154.0** |
| Explain a hash map | **35.8** | | |
| Fifty identical Python clamps | **51.5** | | |
| A JSON blob of fake GPU stats | **32.6** | | |
| `2^10 + 3^5`, integer only | **18.4** | | |

That 154.0 is four streams summed. Wall-clock it was 72.8. The cap again: two run, two wait. Per-stream health held, the system just doesn't go faster past two clients. And this time each lane ran at its own single-stream speed, so the lanes can't be stacked into a fake total either.

The arithmetic answer is 1267. It said 1243. Speed is 18.4. Both are true. Both models I've run this exact cell on have flubbed it at temperature 0, so I'm starting to suspect my probe, not the models. It stays in the table either way.

Asked it to call a weather tool the proper way, not dump XML in the reply. It did. Zero XML in content.

Count job, one stream: **928 joules** for 400 tokens on the two GPU rails. At US residential power (18.34¢/kWh, EIA) that's about **47 cents a day** if you could hold that rate. Same firehose on DeepSeek's own API ($1.20/M output) is about **$4.72/day**. Grok 4.6 output pricing is the frontier ref at **$23.61/day**. That's electricity. The boxes still cost a car.

## The concurrency ladder

Same essay job, unique salt per stream, up to 128 clients. This is the one place I pushed past the cap on purpose.

| Streams | Decode aggregate | Per-stream mean |
|---:|---:|---:|
| 1 | **29.08** | 31.7 |
| 2 | **43.46** | 23.7 |
| 4 | 30.43 | 24.1 |
| 8 | 26.62 | 24.3 |
| 16 | 25.46 | 24.4 |
| 32 | 24.12 | 24.4 |
| 64 | 23.82 | 24.2 |
| 128 | 23.99 | 24.4 |

Peak is at two. Past that you're queuing, and per-stream settles around 24. The single-stream essay here reads 29 against the 35.8 above because this ladder salts every prompt uniquely. Different day, different job, kept apart.

---

## Long context

### Can it still see a needle?

Three codes, planted at 5%, 50%, and 95%. Once per depth.

| I aimed at | Server said | Found | Prefill | Time to first token |
|---:|---:|---|---:|---:|
| 8k | 17,169 | 3/3 | 781 tok/s | 22 s |
| 32k | 69,954 | 3/3 | 779 | 86 s |
| 128k | 273,840 | 3/3 | 726 | 366 s |
| 256k | 547,613 | 3/3 | 662 | 844 s |

**12/12.** Prefill barely droops: 781 down to 662 across a 32x range of context. Retrieval is not a speed chart.

### How fast does it talk after that fill?

Forced 256 tokens so it couldn't quit early.

| I aimed at | Server said | Decode tok/s |
|---:|---:|---:|
| 8k | 16,777 | **32.4** |
| 32k | 67,010 | **35.0** |
| 128k | 267,983 | **32.6** |
| 256k | 559,235 | **30.5** |

Flat. Within 6% from 17k to 559k actual tokens. Whatever decay people expect from long context, this checkpoint doesn't have it.

### Same jobs, cache already full

| Job | ~67k | ~268k | Empty (from above) |
|---|---:|---:|---:|
| Count 1 to 200 | **53.4** | **53.1** | 45.5 |
| Hash map | **34.1** | **35.2** | 35.8 |
| Python clamps | **48.7** | **48.1** | 51.5 |
| Weather tool | called | called | called |

Deeper context didn't cost anything. Counting actually reads faster warm.

---

## Warm session, like an agent already in the middle of work

About 35k already in the prompt. Tools attached. One turn each.

| I asked | Prompt tokens | What it did | tok/s |
|---|---:|---|---:|
| Write a tower-stacking game as one HTML file | 32,379 | Wrote a full 2048 tokens, clean tool call, no XML | **46.9** |
| Look up spaced repetition, then build a flashcard app | 35,056 | 98 tokens of plan, then two web searches | **48.5** |

One wrote a game, one reached for search. Different jobs, not charted against each other. Tower run burned 10,072 joules, about 4.9 per token.

---

## What I didn't run

No thinking-on lanes. No wall-plug power. No vision, though the checkpoint carries a tower. JSON quit-shape quirks from the GLM kit don't apply here because JSON only ran single-stream. I didn't tune DSpark past k=3. The Engram tables (shards 47 and 48, ~95 GiB each, unquantized) made first load slow over NFS. Budget for it.

---

## Credit

Recipe: [MiaAI-Lab/DeepSeek-v4.1-Flash-EXL3-2x-DGX-Sparks](https://github.com/MiaAI-Lab/DeepSeek-v4.1-Flash-EXL3-2x-DGX-Sparks) `@8530568`.
Quant: [Mia-AiLab/DeepSeek-V4.1-Flash-EXL3-2.9bpw](https://huggingface.co/Mia-AiLab/DeepSeek-V4.1-Flash-EXL3-2.9bpw).
Power rate: EIA, US residential. API prices from 14 Sep 2026 (DeepSeek first-party list, $1.20/M out).
