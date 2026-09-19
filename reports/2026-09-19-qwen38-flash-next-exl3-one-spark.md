# Qwen3.8-Flash-Next EXL3 on one DGX Spark

2026-09-19 · [@yume_arasaki](https://x.com/yume_arasaki)

**Verdict: `grid_test_failed`.**

Ran [Cruz's single-Spark EXL3 recipe](https://github.com/vcruz305/Qwen3.8-Flash-Next-EXL3-DGX-Spark-recipe) (`b942e1f`) on one GB10 box. Tensor parallel 1. Weights `turboderp/Qwen3.8-Flash-Next-exl3` rev `3.05bpw_h5_ng5`. Recipe port 8899 remapped to **8009** on this desk (never 8888).

That recipe is **two engines**, not one. I ran the Yume_Arasaki grid on the vLLM overlay because that is the OpenAI `/v1` the grid and Hamster need. His **custom ExLlamaV3** is the faster engine. I did not grid it. Details below.

Cruz's `b942e1f` serve script ships `--reasoning-parser qwen3` and **no** `--tool-call-parser`. The model speaks Qwen3 XML, not Hermes. We patched that ourselves on the live box: `--enable-auto-tool-choice --tool-call-parser qwen3_xml`. First try was `hermes`. That leaked `<function=get_weather><parameter=city>…</parameter></function>` into `content` and left `tool_calls` empty. After the patch, empty-KV weather is a clean `finish_reason=tool_calls`. The patch is **not** in Cruz HEAD. The EngineCore crash still happens with it on.

This is not the [dual NVFP4 write-up](2026-09-18-qwen38-flash-next-nvfp4-two-sparks.md) (`d2f54b7`). Different checkpoint, different engine, different topology, different SHA. `compare_ok` against that isolate is **not comparable**. Printed, never subtracted.

The empty-KV speed cells mostly ran. The grid still fails the thing I actually use these boxes for.

What failed, on this SHA, this pack, this serve:

- Needle **12/15**. The 240k plant is 0/3.
- Decode after fill is flat to ~110k, then **20.9** tok/s at 239k. That is the MTP cliff, not a slow day.
- Tools with the KV already full (32k and 128k) did **not** emit a tool call. Those cells ran **before** we patched the parser. Empty-KV weather later passed, on **our** `qwen3_xml` patch, not on Cruz as shipped.
- Empty-KV **stream** can 200 with zero first token. Night-1 had to go non-stream to get numbers at all.
- A real Hermes tool loop (50–60k boot + tools, `stream=true`, `max_tokens=65536`) killed EngineCore **twice** on a fresh boot. Same `10240×336` cuBLAS `mm`. Process gone.

I didn't mash anything into one score. Counting, writing, code, JSON, tools, and a long-context retrieve are different days at the office. Structured, prose, code, and JSON carried the concurrency axis; the rest ran one stream each.

Tok/s below is after the first token shows up. Divide by the whole wait, prompt-reading included, and you get a different, worse number. Kept apart.

Prompt sizes are what the server said it read. Power is **one** GPU, from `nvidia-smi` on spark-1, while that request was in flight. Not the wall plug. Not both boxes. Not what I paid for the Spark.

---

## Cruz's custom ExLlamaV3

[vcruz305/exllamav3](https://github.com/vcruz305/exllamav3) is not stock turboderp. `master` is upstream plus aarch64 build guards ([#1](https://github.com/vcruz305/exllamav3/pull/1)) plus the GB10 decode work ([#2](https://github.com/vcruz305/exllamav3/pull/2), [#3](https://github.com/vcruz305/exllamav3/pull/3)): int8 GatedResidual mixer kernels (the mixers ship fp16 inside a 3-bit pack), pruned draft `lm_head` (64K-column slice), MTP host-sync removal, wide cooperative MoE tile on GB10's 48 SMs, 8-bit KV in the launcher. Native numbers below are his, on [`523ecd3`](https://github.com/vcruz305/exllamav3/commit/523ecd3), `examples/chat.py`, greedy, 400 new tokens, cold load, one stream. Printed, never subtracted from my vLLM grid.

| Prompt class (Cruz, native) | Decode tok/s | Draft acceptance |
|---|---:|---:|
| Code (nginx log parser) | **79** | 73% |
| DevOps explainer + YAML | **62** | 59% |
| Prose (350-word story) | **53** | 46% |
| No draft, any prompt | 33 | |
| Code, 240k tokens of context | **72** (fp16 KV: 65) | 71% |

He also prints stock exllamav3 1.5.0 at k=3 around **56** on code, and the vLLM overlay at **50–53**. Native is the one that hits 79. Needle on that native launcher is **exact at 240k** and fails at 300k. My vLLM grid missed 234k 0/3. Different engines. Do not mash.

**What I actually booted for the grid:** vLLM **0.29.0** + [vllm-exl3](https://github.com/vcruz305/vllm-exl3) `@08ed1bf` + **exllamav3 1.4.7 built from source** with his aarch64 patch (`tools/patch_exllamav3_aarch64.py` on `exllamav3/exllamav3_ext`). The plugin imports compiled `exllamav3_ext`; the pure-Python wheel is not enough. Checkout on spark-1: `~/src/exllamav3`. MTP k=3, `max_model_len` 262144. That is the vLLM path of his recipe, not the `523ecd3` native launcher.

Native has **no measured OpenAI `/v1`**. No reasoning parser, no tool-call parser, no TabbyAPI numbers on this kit. The grid, the voxel 1-shot, and Hamster all need `/v1`. So `grid_test_failed` is a fail of the **vLLM overlay**, sitting on his kernels, not a fail of the 79 tok/s native engine. I did not run nights through `scripts/exl3_native/`.

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

Asked it to call a weather tool the proper way, not dump XML in the reply. On Cruz as shipped (`hermes`, or no parser): it dumps Qwen3 XML into `content`. **Empty KV, after our `qwen3_xml` patch:** it did. Zero XML in content. `finish_reason=tool_calls`, `get_weather({"city":"Tokyo"})`. That cell is not the crash, and it is not the tools-at-depth cell.

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

**12/15.** Prefill still ~1,070 tok/s at 234k. Retrieval is not a speed chart. The last rung is a miss, not a maybe. Cruz's **native** ExLlamaV3 launcher reports needle exact at 240k. This miss is the vLLM overlay.

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

This is the desk patch. Cruz did not ship it.

His `scripts/serve_one_spark_qwen.sh` `@b942e1f` has `--reasoning-parser qwen3` and stops there. We added `--enable-auto-tool-choice` so OpenAI `/v1` would accept a tools array (without it, tools POSTs 400). First parser we wired was `hermes`. Wrong dialect. The model emits:

```text
<function=get_weather><parameter=city>Tokyo</parameter></function>
```

into `content`. `tool_calls` stays empty. vLLM 0.29 already registers `qwen3_xml` and `qwen3_coder`. We patched the live spark-1 serve (and the Hamster wrapper) from `hermes` to `qwen3_xml`, restarted, re-ran empty-KV weather. Clean call. Isolate `2026-09-19_8009_tools-omp-v1.json`: `tool_calls_present=1`, `xml_in_content=false`.

Night-2 tools-at-depth was already on disk from the `hermes` pass. I did not pretend those rows passed.

| Depth | Parser on the box | Tool call |
|---|---|---|
| empty KV | our `qwen3_xml` patch | clean, no XML |
| 32k resident | Cruz/`hermes` (pre-patch) | **fail** — no tool call |
| 128k resident | Cruz/`hermes` (pre-patch) | **fail** — no tool call |

This is the number I care about for agent work. Parser is now our problem, solved. The engine death on the second generate is not.

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

Hardware for the dump: DGX Spark GB10, aarch64, driver 580.159.03, CUDA 13.0, torch 2.13.0+cu130, UMA 121.69 GiB. Load 79.96 GiB, weights+non-torch 83.02 GiB, KV 11.34 GiB. Kernels: Cruz **exllamav3 1.4.7** `exllamav3_ext` (aarch64-patched) under vllm-exl3, not the native `523ecd3` chat.py launcher.

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
  --tool-call-parser qwen3_xml \  # desk patch; Cruz b942e1f ships none, hermes leaks XML
  --reasoning-parser qwen3 \
  --mamba-ssm-cache-dtype bfloat16 \
  --speculative-config '{"method":"mtp","num_speculative_tokens":3}'
```

Two streaming chat completions, tools on, do not restart between them. Req 1: ~50k+ filler + a tool schema, `stream=true`, `max_tokens=65536`, `temperature=1.0`, `top_p=0.95`, `top_k=20`. Expect 200 and a tool call. Req 2: same messages plus `role=tool` result, immediately. Expect EngineCore death with the `10240×336` `mm` if the bug is still present.

Smaller grid cells (≤35k unique-salt, `max_tokens≤2048`, thinking off, no tool follow-up) did not hit this on this kit.

Ruled out: wrong port, wrong served id, `hermes` parser (we patched that; crash remains with our `qwen3_xml`), dual-node / NCCL (this is TP=1), missing pack, KV too small for 60k.

Likely knobs for Cruz: CUDA-graph + QSA paged kernel + chunked prefill 832 on a 57k–60k cached prefix (capture sizes max 32); warmup for those three QSA kernels at Hermes prompt size; the inductor `mm` shape `(10240, 336)` / `reinterpret_tensor` layout on GB10 bf16; MTP k=3 still on during the 832-token chunk.

---

## What I didn't run

No nights on Cruz's **native** ExLlamaV3 (`523ecd3`, `scripts/exl3_native/`, no `/v1`). That is the engine that prints 79 on code and retrieves a needle at 240k. I used the vLLM overlay because Hamster and the grid speak OpenAI. No TabbyAPI wrap of native. No thinking-on grid lanes (one 1-shot voxel HTML is not a night). No wall-plug power. No concurrency at depth. No 256k job-at-depth — the frozen pair is 32k / 128k, and vLLM 234k already missed the needle. No chart against dual NVFP4 `d2f54b7` or against the old TP=1 NVFP4 `ef1af5f`. Those are other strains.

I did not call this a pass because count-to-200 looks like 65.9. I also did not call Cruz's native engine a fail. I never put the grid on it.

---

## Credit

Recipe: [vcruz305/Qwen3.8-Flash-Next-EXL3-DGX-Spark-recipe](https://github.com/vcruz305/Qwen3.8-Flash-Next-EXL3-DGX-Spark-recipe) `@b942e1f`.  
Custom ExLlamaV3: [vcruz305/exllamav3](https://github.com/vcruz305/exllamav3) `@523ecd3` (aarch64 guards + GB10 decode: int8 GatedResidual, pruned draft `lm_head`, MTP host-sync removal). Native code **79** tok/s is his, not mine. Grid did not run on this path.  
vLLM overlay: [vcruz305/vllm-exl3](https://github.com/vcruz305/vllm-exl3) `@08ed1bf` + **exllamav3 1.4.7 from source** with `tools/patch_exllamav3_aarch64.py`. That is what `:8009` served.  
Quant: [turboderp/Qwen3.8-Flash-Next-exl3](https://huggingface.co/turboderp/Qwen3.8-Flash-Next-exl3) rev `3.05bpw_h5_ng5`.  
Desk patch (ours, not in Cruz HEAD): `--enable-auto-tool-choice --tool-call-parser qwen3_xml`. Cruz `b942e1f` serve script ships no tool parser; `hermes` dumps Qwen3 XML into `content`. (His older GGUF-vs-EXL3 sixcat row already required `qwen3_xml`.)  
Strain: `qwen38-fn-exl3/vllm-tp1/vcruz/b942e1f`. Isolates: `2026-09-19_8009_*.json`.  
Power rate: EIA, US residential. API prices from 19 Sep 2026 (Grok 4.6 first-party list, $6.00/M out).  
Not comparable: [Qwen3.8-Flash-Next NVFP4 on two DGX Sparks](2026-09-18-qwen38-flash-next-nvfp4-two-sparks.md) `@d2f54b7`.
