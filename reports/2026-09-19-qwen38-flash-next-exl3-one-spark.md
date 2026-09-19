# Qwen3.8-Flash-Next EXL3 on one DGX Spark

2026-09-19 · [@yume_arasaki](https://x.com/yume_arasaki)

**Verdict: `grid_test_failed`.**

Ran [Cruz's single-Spark EXL3 recipe](https://github.com/vcruz305/Qwen3.8-Flash-Next-EXL3-DGX-Spark-recipe) (`b942e1f`) plus [vllm-exl3](https://github.com/vcruz305/vllm-exl3) (`08ed1bf`) on one GB10 box. Tensor parallel 1. Weights `turboderp/Qwen3.8-Flash-Next-exl3` rev `3.05bpw_h5_ng5`. vLLM 0.29.0. MTP 3. `max_model_len` 262144. Thinking off for the grid. Recipe port 8899 remapped to **8009** on this desk (never 8888).

This is not the [dual NVFP4 write-up](2026-09-18-qwen38-flash-next-nvfp4-two-sparks.md) (`d2f54b7`). Different checkpoint, different engine, different topology, different SHA. `compare_ok` against that isolate is **not comparable**. Printed, never subtracted.

The empty-KV speed cells mostly ran. The grid still fails the thing I actually use these boxes for.

What failed, on this SHA, this pack, this serve:

- Needle **12/15**. The 240k plant is 0/3.
- Decode after fill is flat to ~110k, then **20.9** tok/s at 239k. That is the MTP cliff, not a slow day.
- Tools with the KV already full (32k and 128k) did **not** emit a tool call. Empty-KV weather later did, after `--tool-call-parser qwen3_xml`.
- Empty-KV **stream** can 200 with zero first token. Night-1 had to go non-stream to get numbers at all.
- A real Hermes tool loop (50–60k boot + tools, `stream=true`, `max_tokens=65536`) killed EngineCore **twice** on a fresh boot. Same `10240×336` cuBLAS `mm`. Process gone.

I didn't mash anything into one score. Counting, writing, code, JSON, tools, and a long-context retrieve are different days at the office. Structured, prose, code, and JSON carried the concurrency axis; the rest ran one stream each.

Tok/s below is after the first token shows up. Divide by the whole wait, prompt-reading included, and you get a different, worse number. Kept apart.

Prompt sizes are what the server said it read. Power is **one** GPU, from `nvidia-smi` on spark-1, while that request was in flight. Not the wall plug. Not both boxes. Not what I paid for the Spark.

---

## Empty context

Two runs at each concurrency, headline is the better one.

| What I asked | 1 stream | 2 streams | 4 streams |
|---|---:|---:|---:|
| Count from 1 to 200 | **65.9** | **112.7** | **202.1** |
| Explain a hash map | **49.4** | **79.5** | **124.9** |
| Fifty identical Python clamps | **61.2** | **99.2** | **170.4** |
| A JSON blob of fake GPU stats | **43.4** | **65.0** | **117.4** |
| `2^10 + 3^5`, integer only | **11.7** | | |

Four is the recipe cap (`MAX_NUM_SEQS=4`). 202.1 is four streams summed on count, on one Spark.

The arithmetic answer is 1267. It said 1283. Speed is 11.7. Both are true. Same wrong 1283 the dual NVFP4 cell printed. At this point it's my probe, not the models. It stays in the table.

Asked it to call a weather tool the proper way, not dump XML in the reply. **Empty KV, after `qwen3_xml`:** it did. Zero XML in content. `finish_reason=tool_calls`. That cell is not the crash, and it is not the tools-at-depth cell.

Count job, one stream: **325 joules** for 400 tokens on the one GPU rail, 0.81 J/tok. At US residential power (18.34¢/kWh, EIA) that's about **24 cents a day** if you could sit on that rate. Same firehose on Grok 4.6 output pricing ($6.00/M, frontier ref) is about **$34.17/day**. That's electricity. The box still costs a car. This is not an agent day.

---

## Long context

Window is 262144, not the dual 1M. Ladder stops where the pack stops.

### Can it still see a needle?

Three codes, planted at 5%, 50%, and 95%. Once per depth. Unique salt.

| I aimed at | Server said | Found | Prefill | Time to first token |
|---:|---:|---|---:|---:|
| 8k | 8,347 | 3/3 | 920 tok/s | 9.1 s |
| 32k | 32,679 | 3/3 | 1,105 | 29.6 s |
| 64k | 65,328 | 3/3 | 1,088 | 60.0 s |
| 110k | 111,700 | 3/3 | 1,066 | 104.8 s |
| 240k | 234,605 | **0/3** | 1,072 | 218.9 s |

**12/15.** Prefill still ~1,070 tok/s at 234k. Retrieval is not a speed chart. The last rung is a miss, not a maybe.

### How fast does it talk after that fill?

Forced 256 tokens so it couldn't quit early.

| I aimed at | Server said | Decode tok/s |
|---:|---:|---:|
| 8k | 7,999 | **43.2** |
| 32k | 33,254 | **44.0** |
| 64k | 67,750 | **43.3** |
| 110k | 113,730 | **43.3** |
| 240k | 239,073 | **20.9** |

Flat through 110k. 239k is half speed. The 234k needle miss and this 20.9 sit on the same cliff.

### Same jobs, cache already full

Frozen depths **32,768** and **131,072**. Eight cells.

| Job | ~32k | ~128k | Empty (from above) |
|---|---:|---:|---:|
| Count 1 to 200 | **66.2** | **63.7** | 65.9 |
| Hash map | **46.1** | **47.4** | 49.4 |
| Python clamps | **63.3** | **62.8** | 61.2 |
| Weather tool | **no call** | **no call** | called (later, empty KV, `qwen3_xml`) |

Count / essay / code barely moved. Tools at depth returned 27 tokens, `finish_reason=stop`, `tool_calls_present=false`. I am not printing 0.89 / 0.21 tok/s as a speed. Those rows are a fail.

---

## The tool lane, where agents live

| Depth | Tool call |
|---|---|
| empty KV, `qwen3_xml` | clean, no XML |
| 32k resident | **fail** — no tool call |
| 128k resident | **fail** — no tool call |

`--tool-call-parser hermes` dumps Qwen3 XML into `content` and leaves `tool_calls` empty. The empty-KV isolate was re-run with `qwen3_xml` and passed. The depth cells were already on disk from the night-2 pass. I did not pretend they passed.

This is the number I care about for agent work. On this SHA it is the failure.

---

## Warm session, like an agent already in the middle of work

About 35k already in the prompt. Tools attached. One turn each. Thinking off. Not the real GUIs.

| I asked | Prompt tokens | What it did | tok/s |
|---|---:|---|---:|
| Write a tower-stacking game as one HTML file | 35,980 | 2,048 tokens of HTML, hit length, no tool call | **30.5** |
| Look up spaced repetition, then build a flashcard app | 37,865 | 94 tokens of plan, `finish_reason=stop`, no tool call | **48.4** |

One wrote until the cap. One stopped. Neither reached for a tool. Different jobs, not charted against each other. Tower run burned 4,299 joules, 2.1 per completion token. Research turn 2,572 joules on 94 tokens — most of that is the 37k read, not the talk.

These 35k one-shots did **not** crash the engine. The crash is the next section.

---

## What died: Hermes tool loop, 2/2

Same serve, same pack, after the grid. Nous Hermes (Hamster / Telegram) on spark-1 `:8009`.

Pattern, twice, cold boot each time:

1. `POST /v1/chat/completions` **stream=true**, ~50–60k prompt (system + 31 tools + history), sampling `temperature=1.0 top_p=0.95 top_k=20`, `max_tokens=65536`.
2. First generate **HTTP 200**. ~220–250 completion tokens. Model emits a tool call.
3. Client runs the tool (~0.3s). Immediate second `POST` with the tool result on the same ~60k prefix.
4. EngineCore JIT of QSA paged kernels (if not already compiled) → **cuBLAS internal error** on a fixed inductor `mm` → illegal memory access → process dead.
5. API 500 `EngineCore encountered an issue`, then connection refused.

Not KV OOM. After turn 1, KV was **37.1%** of 413,829 tokens. 60k prompt fits. UMA after death: ~4 GiB used (weights unloaded). Not a leak. The process exited.

### Incident A — 12:37–12:38

Hermes session `20260919_032150_439778aa`. First generate `in=59261 out=249` in 61.6s, called `terminal`. Tool returned in 0.29s. Second generate died.

Dump on the dying request: `prompt_token_ids_len=60057`, `num_computed_tokens=57408`, scheduled chunk **832**, CUDA graphs `FULL_AND_PIECEWISE` with capture sizes `[1, 2, 4, 8, 16, 24, 32]`.

### Incident B — 12:55–12:58

Fresh `sparks up`. First generate `in=50292 out=224` in 52.6s. Same `mm` crash.

### Exact exception (identical in A and B)

```text
extern_kernels.mm(buf2, reinterpret_tensor(arg3_1, (10240, 336), (1, 10240), 0), out=buf3)
RuntimeError: CUDA error: CUBLAS_STATUS_INTERNAL_ERROR when calling
  cublasGemmEx(..., a CUDA_R_16BF lda, b CUDA_R_16BF ldb, c CUDA_R_16BF or CUDA_R_32F, CUBLAS_GEMM_DEFAULT_TENSOR_OP)
```

Then `torch.AcceleratorError: CUDA error: an illegal memory access was encountered`.

vLLM also warns, on the first Hermes-sized prefill, not on empty-KV grid cells:

```text
Triton kernel JIT compilation during inference: _expand_qsa_indices_kernel
Triton kernel JIT compilation during inference: _qsa_mqa_paged_kernel
Triton kernel JIT compilation during inference: _qsa_sparse_paged_gqa_splitk_kernel
This causes a latency spike; consider extending warmup to cover this shape/config.
```

Hardware for the dump: DGX Spark GB10, aarch64, driver 580.159.03, CUDA 13.0, torch 2.13.0+cu130, UMA 121.69 GiB. Load 79.96 GiB, weights+non-torch 83.02 GiB, KV 11.34 GiB.

### How to reproduce without Hermes

Cold boot:

```bash
vllm serve /home/yum-spark-1/models/Qwen3.8-Flash-Next-EXL3 \
  --served-model-name Qwen3.8-Flash-Next-EXL3 \
  --host 0.0.0.0 --port 8009 \
  --quantization exl3 \
  --max-model-len 262144 \
  --max-num-seqs 4 \
  --gpu-memory-utilization 0.80 \
  --enable-prefix-caching \
  --trust-remote-code \
  --enable-auto-tool-choice \
  --tool-call-parser qwen3_xml \
  --reasoning-parser qwen3 \
  --mamba-ssm-cache-dtype bfloat16 \
  --speculative-config '{"method":"mtp","num_speculative_tokens":3}'
```

Two streaming chat completions, tools on, do not restart between them. Req 1: ~50k+ filler + a tool schema, `stream=true`, `max_tokens=65536`, `temperature=1.0`, `top_p=0.95`, `top_k=20`. Expect 200 and a tool call. Req 2: same messages plus `role=tool` result, immediately. Expect EngineCore death with the `10240×336` `mm` if the bug is still present.

Smaller grid cells (≤35k unique-salt, `max_tokens≤2048`, thinking off, no tool follow-up) did not hit this on this kit.

Ruled out: wrong port, wrong served id, `hermes` parser (crash remains with `qwen3_xml`), dual-node / NCCL (this is TP=1), missing pack, KV too small for 60k.

Likely knobs for Cruz: CUDA-graph + QSA paged kernel + chunked prefill 832 on a 57k–60k cached prefix (capture sizes max 32); warmup for those three QSA kernels at Hermes prompt size; the inductor `mm` shape `(10240, 336)` / `reinterpret_tensor` layout on GB10 bf16; MTP k=3 still on during the 832-token chunk.

---

## What I didn't run

No thinking-on grid lanes (one 1-shot voxel HTML is not a night). No wall-plug power. No concurrency at depth. No 256k job-at-depth — the frozen pair is 32k / 128k, and 234k already missed the needle. No chart against dual NVFP4 `d2f54b7` or against the old TP=1 NVFP4 `ef1af5f`. Those are other strains.

I did not call this a pass because count-to-200 looks like 65.9.

---

## Credit

Recipe: [vcruz305/Qwen3.8-Flash-Next-EXL3-DGX-Spark-recipe](https://github.com/vcruz305/Qwen3.8-Flash-Next-EXL3-DGX-Spark-recipe) `@b942e1f`.  
Plugin: [vcruz305/vllm-exl3](https://github.com/vcruz305/vllm-exl3) `@08ed1bf`.  
Quant: [turboderp/Qwen3.8-Flash-Next-exl3](https://huggingface.co/turboderp/Qwen3.8-Flash-Next-exl3) rev `3.05bpw_h5_ng5`.  
Strain: `qwen38-fn-exl3/vllm-tp1/vcruz/b942e1f`. Isolates: `2026-09-19_8009_*.json`.  
Power rate: EIA, US residential. API prices from 19 Sep 2026 (Grok 4.6 first-party list, $6.00/M out).  
Not comparable: [Qwen3.8-Flash-Next NVFP4 on two DGX Sparks](2026-09-18-qwen38-flash-next-nvfp4-two-sparks.md) `@d2f54b7`.
