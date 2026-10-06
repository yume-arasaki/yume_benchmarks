# CYBER-FROST-3.8 on one DGX Spark (cruz's recipe, full agent grid)

2026-10-07 · [@yume_arasaki](https://x.com/yume_arasaki)

cruz's [DGX Spark recipe](https://github.com/vcruz305/Qwen3.8-Flash-Next-EXL3-DGX-Spark-recipe) for EXL3 packs — his exllamav3 fork with mixed-K cooperative decode kernels, served through TabbyAPI — rebuilt from his DEPLOY.md at his exact pins, then the full agent grid on top of it. Same base model as the Sep 20 grids, different checkpoint: `cyberfrost-sage-exl3/tabby-coopmk/vcruz/047ce72`, the SAGE 3.87bpw EXL3 quant of CYBER-FROST-3.8 (de-refused fine-tune of Qwen3.8-Flash-Next 180B — what the fine-tune actually does is a separate question, covered in the template A/B report).

Strain: `cyberfrost-sage-exl3/tabby-coopmk/vcruz/047ce72`. Fork pin 047ce72 (nvcc sm_121, ~22 min compile), TabbyAPI pin 7a524a9, `EXL3_MOE_COOP_MIXEDK=1 EXL3_MOE_MIXEDK_NOSYNC=1`, `NGRAM_RAM=false`, MTP depth 5, thinking-off view per his DEPLOY.md (probe-verified: zero reasoning tokens with `enable_thinking: false`). Port 8012, one GB10. Frozen grid scripts, temperature 0, unique salt per stream.

---

## The build

Two env vars flip the MoE decode path from unified to cooperative mixed-K. The log line you want is "Mixed-K coop decode kernels" — 40 of them at load. If that line is missing you are on the slow path and do not know it.

Two traps, both real: TabbyAPI main is currently broken against this fork (a version check landed Oct 5 demanding exllamav3 1.5.4; the fork measures 1.5.1) — pin `7a524a9`. And `serve.sh` defaults `NGRAM_RAM=true` while his DEPLOY.md says false; his env streams the 32 GB n-gram table from disk and frees ~30 GB. Use his env, not the script default.

## bench_v1 (his bench, his parameters)

400 tokens, greedy, thinking on, median of 3, one Spark.

| Config | Mine | His claim |
|---|---:|---:|
| Coop kernels ON | 60.7 tok/s (best 64.1) | 53.2 |
| Coop kernels OFF | 42.7 tok/s | 37.0 |
| Delta | +42% | +44% |

Both his numbers verified and slightly exceeded on my hardware. TensorFold 0.6.0 stock kernels on the same pack: 57.3 — the coop path still wins.

## Empty context (two runs a lane, headline = peak; comparison column = mean)

| Lane | C1 | C2 | C4 | Sep 20 cousin (qfn 523ecd3, C1) | Delta |
|---|---:|---:|---:|---:|---:|
| Count 1 to 200 (structured) | 85.1 | 85.0 | 85.1 | 106.2 | −22% |
| Python clamps (code) | 82.5 | 83.5 | 84.0 | 102.3 | −20% |
| Explain a hash map (prose) | 51.7 | 52.9 | 53.9 | 71.1 | −30% |
| JSON GPU stats | 41.4 | 43.9 | 41.9 | 65.8 | −38% |

Concurrency is flat C1→C4 — the coop kernels keep batch scaling honest (cousin: 107.7 / 107.3).

## Why it is slower (the arithmetic)

The recipe is not at fault. The quant is. SAGE pack body runs 4.0 bpw (bits 4, head_bits 8, vision_bits 6, mtp_bits 4) vs the cousin's 3.05 bpw body with 5-bit non-routed overrides. Both carry the same ~32 GB n-gram table: bodies are ~62.5 vs ~47.5 GB. 6B active params per token → 3.0 vs 2.29 GB of weight traffic = 1.31x bytes, which predicts about −24% on a bandwidth-bound GB10. Measured −20/−22/−30. The json overshoot (−38%) is 65-token completions that cannot amortize the ~9 KB jailbreak template the pack's jinja renders into every call; prose −30% tracks mediocre MTP acceptance on the fine-tune's shifted distribution (~45% in the smoke sample). cruz kept headroom on purpose — 8-bit heads, 6-bit vision — so the de-refusal fine-tune survives compression. The cousin is a diet pack; this is the armor pack.

## Needle, context ladder, agent workloads

- Needle 3/3 at every depth, 8k to 243k actual (15/15 across depths × positions). No degradation.
- Context decode ladder: 47.3 / 40.5 / 46.7 / 44.8 / 39.9 tok/s (8k → 243k). Prefill 793-995 tok/s.
- 35k-token Hermes agent workload: clean `web_search` tool call, no XML-in-content, 60.3 tok/s decode, 37,934 prompt tokens. Generate turns: 68.5 / 60.7 (cousin 86.8 / 88.3).
- 35k OMP tower: 32.4 tok/s sustained on a 2,048-token completion.
- Tools lane decode numbers are excluded (one-chunk delivery makes them divide-by-~0 artifacts; `tool_calls_present` is the meaningful signal and it is clean).

## Scope

One Spark, one pack, decode throughput and agent behavior. No quality evals, no perplexity, quant fidelity unmeasured. Raw isolates: `runs/2026-10-07/` (13 files, `2026-10-07_8012_*`). Template A/B on the same checkpoint: `cf-scoped-test-2026-10-06.md`.
