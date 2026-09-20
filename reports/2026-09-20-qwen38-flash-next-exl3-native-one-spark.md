# Qwen3.8-Flash-Next EXL3 native on one DGX Spark (recipe speed)

2026-09-20 · [@yume_arasaki](https://x.com/yume_arasaki)

Same pack, same fork, same port as the [19 Sep native write-up](2026-09-19-qwen38-flash-next-exl3-native-one-spark.md). That run was **not** Cruz's launcher to the tee: the `/v1` shim iterated off-thread, no `torch.inference_mode`, cache loaded at batch 4. A greedy 400-token code job on that shim was **39.6 tok/s**. His published number is **79**.

This run is the fix. Generator on the request thread, `inference_mode`, batch 1, flags copied from `scripts/exl3_native/tuning/run-qwen38-exl3.sh`:

`-mtp -ndt 5 -dds -dc 0.6 -cq 8,8 -cs 262144` plus `EXL3_GR_INT8=1 EXL3_MOE_COOP_WIDE=1 EXL3_INT8_GEMV=0`, big-core `taskset`, pack native-view, [vcruz305/exllamav3](https://github.com/vcruz305/exllamav3) `@523ecd3`. Port **8009**.

A Cruz-style greedy 400-token nginx-parser job on this serve: wall **79.5 tok/s**, engine **83.8**, draft accept **74%**. That is his 79 / 73%. The grid below is still the Yume_Arasaki jobs, thinking off, unique salt. Not 79 subtracted from 107.

The [vLLM overlay](2026-09-19-qwen38-flash-next-exl3-one-spark.md) (`grid_test_failed`) is a different engine. `compare_ok` fails. Printed, never subtracted.

C2/C4 do **not** scale. Cruz's Generator is one job at a time, same as `chat.py`. Four streams serialize, so the aggregate sits on C1. I am not pretending this is vLLM C4.

Tok/s is after the first token except where I say wall. Prompt sizes are what the server said it read. Power is one GPU on spark-1.

---

## Empty context

Two runs, headline is the better one. Smoke: 17 × 19 → **323**.

| What I asked | 1 stream | 2 streams | 4 streams |
|---|---:|---:|---:|
| Count from 1 to 200 | **107.1** | 107.7 | 107.3 |
| Explain a hash map | **71.7** | 73.2 | 72.3 |
| Fifty identical Python clamps | **102.6** | 103.8 | 103.9 |
| A JSON blob of fake GPU stats | **66.2** | 66.2 | 65.0 |
| `2^10 + 3^5`, integer only | **14.8** | | |

C2/C4 ≈ C1 is the lock, not a measurement error. Count drafts at **99%** accept; that is why 107 beats his 79 on a different code prompt. His **53** prose is his 350-word story through `chat.py`. My **71.7** is the hash-map 600 on this grid. Same fork, same SHA 523ecd3. Different job. Printed, never subtracted.

The arithmetic answer is 1267. It said 1283. Speed is 14.8. Both are true.

Empty-KV weather: `get_weather`, `finish_reason=tool_calls`, no XML in content.

Count job, one stream: **216 joules** for 400 tokens, 0.54 J/tok. At US residential (18.34¢/kWh, EIA) about **25 cents a day** on that rate. Grok 4.6 output ($6.00/M) on the same firehose is about **$55.51/day**. Electricity. The box still costs a car.

---

## Long context

### Needle

Three codes at 5 / 50 / 95%. Unique salt.

| I aimed at | Server said | Found | Prefill | Time to first token |
|---:|---:|---|---:|---:|
| 8k | 8,205 | 3/3 | 851 tok/s | 9.6 s |
| 32k | 33,921 | 3/3 | 1,011 | 33.6 s |
| 64k | 67,801 | 3/3 | 1,069 | 63.4 s |
| 110k | 111,715 | 3/3 | 1,047 | 106.7 s |
| 240k | 243,635 | **3/3** | 1,043 | 233.7 s |

**15/15.** Including 240k. Cruz's native needle is exact at 240k. This grid agrees.

### Decode after fill

Forced 256 tokens.

| I aimed at | Server said | Decode tok/s |
|---:|---:|---:|
| 8k | 8,165 | **51.5** |
| 32k | 32,651 | **55.6** |
| 64k | 67,762 | **67.4** |
| 110k | 109,606 | **66.0** |
| 240k | 243,596 | **55.8** |

No 20.9 cliff. 240k is still in the 50s.

### Same jobs, cache already full

| Job | ~32k | ~128k | Empty |
|---|---:|---:|---:|
| Count 1 to 200 | **112.9** | **82.0** | 107.1 |
| Hash map | **52.2** | **49.0** | 71.7 |
| Python clamps | **111.1** | **69.0** | 102.6 |
| Weather tool | **called** | **called** | called |

Tools at depth call. I am not printing the 3k tok/s on a 17-token tool after a 33s prefill.

---

## Warm session

First night, tools on, one turn. Prefill **980** / **973**. The shim holds XML until eos, so I am not printing 17k–386k decode on 57–58 tokens.

| I asked | Prompt tokens | What it did |
|---|---:|---|
| Tower game as one HTML file | 35,649 | **tool call** (`bash`), 58 tokens |
| Spaced-repetition app | 37,482 | **tool call** (`web_search`), 57 tokens |

Then a follow-up with tools off, 2048 streamed HTML, `ignore_eos`. That is the real decode. Same serve, same strain.

| I asked | Prompt tokens | Decode tok/s | Engine | Draft |
|---|---:|---:|---:|---:|
| Same tower, write the HTML | 36,578 | **86.8** | 86.7 | 86% |
| Same flashcard app, write the HTML | 36,881 | **88.3** | 88.2 | 87% |

Both hit `length` at 2048. Prefill 1,040 / 1,018. Wall ~35 tok/s because the first token still waits ~35 s. Tower preview is `<title>Tower Stack</title>`. Flashcard is FSRS-lite in one file. GPU rail: 3,653 J / 1.78 J/tok and 3,715 J / 1.81 J/tok.

The tower's first generate turn (tools still on, max_tokens=256) hit `length` with **no** tool call. Hermes's first generate turn did call `web_search` (43 tokens). Those are not the 86.8 / 88.3 rows. The original tool-call isolates stay on disk.

---

## What I didn't run

No thinking-on grid. No wall-plug. No true concurrent decode — the recipe Generator does not batch like vLLM. No TabbyAPI. No chart against overlay 55.1 or dual NVFP4 `d2f54b7`.

Hermes on this serve survived an 80k tool loop (the overlay died). It also emitted `<|im_start|>user` as a 16-character reply because Cruz's stop list is only `<|im_end|>`. That is a template leak, not a crash.

---

## Credit

Custom ExLlamaV3: [vcruz305/exllamav3](https://github.com/vcruz305/exllamav3) `@523ecd3`.  
Recipe launcher: [vcruz305/Qwen3.8-Flash-Next-EXL3-DGX-Spark-recipe](https://github.com/vcruz305/Qwen3.8-Flash-Next-EXL3-DGX-Spark-recipe) `scripts/exl3_native/tuning/run-qwen38-exl3.sh`.  
Quant: [turboderp/Qwen3.8-Flash-Next-exl3](https://huggingface.co/turboderp/Qwen3.8-Flash-Next-exl3) rev `3.05bpw_h5_ng5`.  
Strain: `qwen38-fn-exl3/exllamav3-native/vcruz/523ecd3`. Isolates: `2026-09-20_8009n_*.json`. Generate follow-up: `2026-09-20_8009n_omp-workload-35k_generate.json`, `2026-09-20_8009n_hermes-workload-35k_generate.json`.  
Slow shim (kept): [19 Sep native](2026-09-19-qwen38-flash-next-exl3-native-one-spark.md). Overlay fail: [19 Sep `grid_test_failed`](2026-09-19-qwen38-flash-next-exl3-one-spark.md).  
Power: EIA US residential. API: Grok 4.6 $6.00/M out, 20 Sep 2026.
