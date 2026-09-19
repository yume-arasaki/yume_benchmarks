# Qwen3.8-Flash-Next EXL3 native on one DGX Spark

2026-09-19 · [@yume_arasaki](https://x.com/yume_arasaki)

Ran [Cruz's custom ExLlamaV3](https://github.com/vcruz305/exllamav3) (`523ecd3`) on one GB10 box. Same pack as the [vLLM overlay write-up](2026-09-19-qwen38-flash-next-exl3-one-spark.md) (`grid_test_failed`): `turboderp/Qwen3.8-Flash-Next-exl3` rev `3.05bpw_h5_ng5`. Recipe [vcruz305/Qwen3.8-Flash-Next-EXL3-DGX-Spark-recipe](https://github.com/vcruz305/Qwen3.8-Flash-Next-EXL3-DGX-Spark-recipe) `b942e1f`. Port **8009** (never 8888/8899).

This is **not** that overlay. Different engine. `compare_ok` fails on purpose. Printed, never subtracted.

His fork is upstream plus aarch64 guards plus the GB10 decode work: int8 GatedResidual mixers, pruned draft `lm_head`, MTP host-sync removal, wide cooperative MoE tile, 8-bit KV. Launcher knobs: `-mtp -ndt 5 -dds -dc 0.6 -cq 8,8 -cs 262144`, `EXL3_GR_INT8=1`, `EXL3_MOE_COOP_WIDE=1`, `EXL3_INT8_GEMV=0`, big-core `taskset`. Native `chat.py` has no OpenAI `/v1`. I wrapped his Generator so Hamster and this grid could talk to the same port. Qwen3 XML is parsed into `tool_calls` on the desk (Cruz `b942e1f` serve script still ships no tool parser).

He prints **79** tok/s on a code prompt through `chat.py`. Mine below are the Yume_Arasaki jobs, thinking off, unique salt. Different protocol. I am not subtracting 79 from 54.1.

Tok/s is after the first token. Prompt sizes are what the server said it read. Power is **one** GPU on spark-1.

---

## Empty context

Two runs at each concurrency, headline is the better one. Smoke check first: 17 × 19 → **323**. Correct.

| What I asked | 1 stream | 2 streams | 4 streams |
|---|---:|---:|---:|
| Count from 1 to 200 | **55.1** | **81.3** | **125.9** |
| Explain a hash map | **33.9** | **52.1** | **70.8** |
| Fifty identical Python clamps | **54.1** | **82.0** | **125.1** |
| A JSON blob of fake GPU stats | **32.3** | **58.3** | **73.3** |
| `2^10 + 3^5`, integer only | **14.7** | | |

The arithmetic answer is 1267. It said 1051. Speed is 14.7. Both are true. Probe stays in the table.

Asked it to call a weather tool the proper way, not dump XML in the reply. Empty KV: it did. `get_weather`, `finish_reason=tool_calls`, zero XML left in content.

Count job, one stream: **270 joules** for 400 tokens on the one GPU rail, 0.67 J/tok. At US residential power (18.34¢/kWh, EIA) that's about **16 cents a day** sitting on that rate. Same firehose on Grok 4.6 output pricing ($6.00/M) is about **$28.57/day**. Electricity. The box still costs a car.

---

## Long context

Window is 262144. Cruz's native needle is exact at 240k and fails at 300k. I ran the same ladder the overlay missed.

### Can it still see a needle?

Three codes, planted at 5%, 50%, and 95%. Once per depth. Unique salt.

| I aimed at | Server said | Found | Prefill | Time to first token |
|---:|---:|---|---:|---:|
| 8k | 8,510 | 3/3 | 719 tok/s | 11.8 s |
| 32k | 32,691 | 3/3 | 981 | 33.3 s |
| 64k | 66,570 | 3/3 | 1,014 | 65.7 s |
| 110k | 113,781 | 3/3 | 1,028 | 110.7 s |
| 240k | 243,634 | **3/3** | 1,024 | 237.8 s |

**15/15.** Including 240k. The overlay on this pack was 0/3 at 234k. That was the vLLM path, not the pack.

### How fast does it talk after that fill?

Forced 256 tokens.

| I aimed at | Server said | Decode tok/s |
|---:|---:|---:|
| 8k | 8,472 | **28.7** |
| 32k | 33,267 | **29.1** |
| 64k | 66,531 | **34.2** |
| 110k | 107,539 | **33.9** |
| 240k | 243,597 | **27.9** |

No MTP cliff at 239k. Overlay decode at that depth was 20.9. This stays in the high 20s.

### Same jobs, cache already full

Frozen depths **32,768** and **131,072**. Eight cells.

| Job | ~32k | ~128k | Empty (from above) |
|---|---:|---:|---:|
| Count 1 to 200 | **57.1** | **55.5** | 55.1 |
| Hash map | **32.4** | **31.6** | 33.9 |
| Python clamps | **46.3** | **53.6** | 54.1 |
| Weather tool | **called** | **called** | called |

Tools at depth were the overlay's fail. Here they call at both rungs, no XML in content.

---

## The tool lane, where agents live

| Depth | Tool call |
|---|---|
| empty KV | clean |
| 32k resident | **98.7** tok/s, clean |
| 128k resident | **40.9** tok/s, clean |

Parser is ours, on the shim, not in Cruz HEAD. The 35k Hermes turn also emitted a `web_search` tool call. The 35k tower turn started a `write_file` and hit the 2048 cap before `</function>`, so that row is a long HTML dump, not a clean call. Recorded that way.

---

## Warm session, like an agent already in the middle of work

About 35k already in the prompt. Tools attached. One turn each. Thinking off.

| I asked | Prompt tokens | What it did | tok/s |
|---|---:|---|---:|
| Write a tower-stacking game as one HTML file | 36,306 | 2,048 tokens, started a write_file, hit length | **46.3** |
| Look up spaced repetition, then build a flashcard app | 37,482 | 49 tokens, then a clean `web_search` | **26.2** |

Different jobs. Tower run 4,120 joules, 2.0 per completion token. Research turn 2,466 joules on 49 tokens.

---

## What I didn't run

No thinking-on grid. No wall-plug. No concurrency at depth. No TabbyAPI wrap (Cruz has not qualified that on this pack). No chart against the vLLM overlay or against dual NVFP4 `d2f54b7`. His `chat.py` 79 on code is his protocol, not this table.

The EngineCore `10240×336` cuBLAS crash lived on the vLLM overlay. This serve is not that process.

---

## Credit

Custom ExLlamaV3: [vcruz305/exllamav3](https://github.com/vcruz305/exllamav3) `@523ecd3`.  
Recipe: [vcruz305/Qwen3.8-Flash-Next-EXL3-DGX-Spark-recipe](https://github.com/vcruz305/Qwen3.8-Flash-Next-EXL3-DGX-Spark-recipe) `@b942e1f`.  
Quant: [turboderp/Qwen3.8-Flash-Next-exl3](https://huggingface.co/turboderp/Qwen3.8-Flash-Next-exl3) rev `3.05bpw_h5_ng5`.  
Desk shim: OpenAI `/v1` on **8009** over his Generator, plus Qwen3 XML → `tool_calls`.  
Strain: `qwen38-fn-exl3/exllamav3-native/vcruz/523ecd3`. Isolates: `2026-09-19_8009n_*.json`.  
Overlay (failed, kept): [Qwen3.8-Flash-Next EXL3 on one DGX Spark (`grid_test_failed`)](2026-09-19-qwen38-flash-next-exl3-one-spark.md).  
Power rate: EIA, US residential. API prices from 19 Sep 2026 (Grok 4.6 first-party list, $6.00/M out).
