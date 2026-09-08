# GLM-5.3-Flash EXL3 on two DGX Sparks

2026-09-09 · [@yume_arasaki](https://x.com/yume_arasaki)

Ran [Mia's dual-Spark EXL3 recipe](https://github.com/MiaAI-Lab/GLM-5.3-Flash-EXL3-2x-DGX-Sparks) (`9c0794b`) on two GB10 boxes. Tensor parallel 2. Weights `Mia-AiLab/GLM-5.3-Flash-EXL3-TR3-4bpw`. Drafter on, k=7. Thinking off.

Her public number for count-to-200 is **62.9** tok/s. I got **66.4**. Same kind of job, not the same pin, not the same desk. I'm not going to print a percent. That would be fake precision.

I also didn't mash everything into one score. Count, essay, code, tools, and a long-context retrieve are different days at the office. If I only posted 66.4 you'd think the box talks that fast when you're actually using it. It doesn't.

Tok/s below is after the first token shows up. If you divide by the whole wait, including the model reading the prompt, you get a different (worse) number. I kept them apart.

Prompt sizes are what the server said it read. I aimed at 8k / 32k / 128k / 256k. This tokenizer overshoots. I report the counted size.

Power is both GPUs added together, from `nvidia-smi`, while that request was in flight. Not the wall plug. Not what I paid for the Sparks.

---

## Empty context

Two runs on the count job at 1, 2, and 4 streams. I took the better of the two. Everything else is single stream. I didn't bother sweeping concurrency on the essay. It's slower and I already knew that.

| What I asked | 1 stream | 2 streams | 4 streams |
|---|---:|---:|---:|
| Count from 1 to 200 | **66.4** | **133.4** | **187.6** |
| Explain a hash map | **28.5** | | |
| Fifty identical Python clamps | **61.3** | | |
| A JSON blob of fake GPU stats | **44.7** | | |
| `2^10 + 3^5`, integer only | **29.9** | | |

The arithmetic answer is 1267. It didn't print 1267. Speed is still 29.9. Both are true.

Asked it to call a weather tool the proper way, not dump XML in the reply. It did.

Count job, one stream: **417 joules** for 400 tokens on the two GPU rails. At US residential power (18.34¢/kWh, EIA June 2026) that's about **thirty cents a day** if you could sit on that rate. The same token firehose on Grok 4.6 output pricing is about **$34/day**. GLM-5.3-Flash's own API list is about **$2.87/day**. That's electricity. The boxes still cost a car.

---

## Long context

I stuffed the prompt with filler, then asked it to do something. Three different somethings, because they lie if you only run one.

### Can it still see a needle?

Three codes, planted at 5%, 50%, and 95%. Once per depth. Not the "15/15" trick of repeating 32k five times.

| I aimed at | Server said | Found | Prefill | Time to first token |
|---:|---:|---|---:|---:|
| 8k | 17,968 | 3/3 | 1,483 tok/s | 12 s |
| 32k | 69,454 | 3/3 | 1,615 | 43 s |
| 128k | 279,170 | 3/3 | 1,549 | 180 s |
| 256k | 557,116 | 3/3 | 1,458 | 382 s |

**12/12.** That's "did it find the codes." It is not a speed chart.

### How fast does it talk after that fill?

Forced 256 tokens so it couldn't quit early. This is the curve people actually mean by fall-off. Not the 66.4 from an empty prompt.

| I aimed at | Server said | Decode tok/s |
|---:|---:|---:|
| 8k | 17,208 | **21.6** |
| 32k | 67,967 | **21.8** |
| 128k | 284,962 | **24.3** |
| 256k | 557,086 | **24.7** |

It didn't get slower. It got a little faster. I wasn't expecting that.

### Same jobs, cache already full

| Job | ~32k | ~128k | Empty (from above) |
|---|---:|---:|---:|
| Count 1 to 200 | **65.7** | **65.3** | 66.4 |
| Hash map | **28.2** | **30.2** | 28.5 |
| Python clamps | **61.6** | **63.6** | 61.3 |
| Weather tool | called | called | called |

The real jobs barely moved. The tool rows are twelve tokens. Don't read that as "tools are slow."

---

## Warm session, like an agent already in the middle of work

About 35k already in the prompt. Tools attached. Still not the actual OMP or Hermes apps. One turn each.

| I asked | Prompt tokens | What it did | tok/s |
|---|---:|---|---:|
| Write a tower-stacking game as one HTML file | 34,080 | Wrote a full 2048 tokens, then tried to save a file | **43.8** |
| Look up spaced repetition, then build a flashcard app | 35,298 | 55 tokens of "I'll research first," then two web searches | n/a |

I am not putting 43.8 next to that second turn and calling it a comparison. One of them wrote a game. The other reached for search. That's the whole point of measuring it this way.

---

## What I didn't run

I didn't sweep 2 and 4 streams on the essay. I didn't put maths or JSON at 32k. I didn't time the real GUIs, or thinking-on, or wall power. This kit doesn't give me a clean wall-joule number the way it gives GPU draw.

---

## Credit

Recipe: [MiaAI-Lab/GLM-5.3-Flash-EXL3-2x-DGX-Sparks](https://github.com/MiaAI-Lab/GLM-5.3-Flash-EXL3-2x-DGX-Sparks) `@9c0794b`.  
Power rate: EIA, US residential, June 2026. API prices from 9 Sep 2026.
