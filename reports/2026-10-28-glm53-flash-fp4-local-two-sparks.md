# GLM-5.3-Flash at 4 bits on two DGX Sparks — NVIDIA's own checkpoint, 0.8 off BF16 (2026-10-28)

2026-10-28 · [@yume_arasaki](https://x.com/yume_arasaki)

NVIDIA shipped a local-first checkpoint of GLM-5.3-Flash: 320B MoE, 18B active, NVFP4 weights
recovered with quantization-aware distillation, ~166 GiB, validated by them on exactly my hardware —
two DGX Sparks. [vr8vr8's cluster recipe](https://github.com/myllmbox/glm53-flash-cluster-recipe)
puts it on the pair in one script, TP=2 over RoCE, 16 seats, 400k window. I ran it for one night
looking for the damage 4-bit is supposed to do. It never showed.

The quality argument is NVIDIA's, not mine. Their card's table for this exact checkpoint: GPQA
Diamond **91.3 at 4 bits against 92.1 at BF16**. ToolEvalBench **92.4 against 91.9** — the 4-bit
build out-scored its full-precision parent on tool calling. Their QAD report calls it "consistent
recovery to near-BF16 accuracy." My job was to check whether a fleet full of agent work agrees.

**Strain:** `glm53-flash-fp4-local/vllm-mbx/vr8vr8/76348ddb`
**Recipe:** [myllmbox/glm53-flash-cluster-recipe](https://github.com/myllmbox/glm53-flash-cluster-recipe) `@a60a023` (v3), image `myllmbox/glm53-flash-cluster-vllm:v3`
**Checkpoint:** `nvidia/GLM-5.3-Flash-FP4-Local` @ `76348ddb` — 177.5 GB verified on both nodes
**Engine:** vLLM 0.32 + MBX loader/kernels, TP=2 over RoCE
**Serve:** spark-1 **:8008**, served id `zai-org/GLM-5.3-Flash`, 16 seats, window 400k, KV pool 12G per box
**Install notes:** the desk's — `cluster.env` hand-written (setup.sh auto-discovery picked the WiFi
link, wrong here: forced to `enp1s0f1np1` / `10.0.0.x` / `rocep1s0f1`); `models_dir` and `cache_dir`
absolute under `/srv/glm53-cluster` because the two nodes have different users; port 8008, host
0.0.0.0 for the tailnet. Boot to health 443 s. Lane committed live 26 Oct, 1017 stopped.

## Headline

| | |
|---|---|
| GPQA Diamond | **91.3** @ 4 bits vs 92.1 BF16 (NVIDIA's card) |
| ToolEvalBench | **92.4** @ 4 bits vs 91.9 BF16 — 4-bit wins |
| Needle retrieval | **12/12** at 8k · 32k · 128k · 256k |
| Garbled streams | **0**, every lane |
| 16-stream aggregate | **165.4 tok/s** (sweep clock) |
| Decode at 232k actual | **48.0 tok/s** — rises with context |

## Fidelity in use

The symptoms of a bad quant, for anyone who has run one: reasoning chains that double back, tool
calls arriving with broken JSON, prose repeating at depth. Across every workload I ran on this
checkpoint, none of it appeared.

| Check | Result |
|---|---|
| Garbled streams | zero, every lane |
| Tool calls | parsed clean, no XML fragments in content |
| Needle retrieval | 12/12 at 8k, 32k, 128k, 256k |
| Maths | exact at 1267 tokens of working |
| Window | requests past 400k refused at the door, no silent truncation |

Retrieval at depth is where quants fall apart first, so the 12/12 line is the one I trust most.

**No BF16 A/B.** Two Sparks cannot hold a 320B model at BF16, so there is no same-rig comparison to
make. The quality case above is NVIDIA's table; the receipts below are mine; they agree, and that
is as far as the claim goes.

## Speed, two clocks

Night 1: empty KV cache, temperature zero. Aggregate decode, tok/s by stream count:

| Lane | 1 stream | 2 streams | 4 streams |
|---|---:|---:|---:|
| Structured (count 1→200) | **54.6** | 88.8 | 145.7 |
| Prose 600 | 42.3 | 62.4 | 84.9 |
| Code clamps | 61.5 | 98.0 | 157.2 |

Concurrency sweep: loaded pool, its own clock, a different night. Aggregate, per-stream,
GPU-rail joules per token:

| Streams | Aggregate | Per-stream | J/token |
|---:|---:|---:|---:|
| 1 | 37.9 | 42.3 | 1.97 |
| 4 | 80.4 | 22.1 | 1.00 |
| 8 | 111.7 | 15.2 | 0.76 |
| 16 | **165.4** | 11.2 | **0.55** |

Stops at 16 because that is `max_num_seqs`. Energy per token drops 3.6x from C1 to C16 — the
number that matters if you bill by work done. This is the first GLM lane on this pair that keeps
scaling past four streams; the TensorFold lanes queue above four seats. Different engines, never
subtracted.

## Depth

Decode is supposed to fall as context grows. Here it rises, tok/s by actual tokens:

| Aimed | Actual | Decode tok/s |
|---|---:|---:|
| 8k | 8k-class | 30.3 |
| 32k | 32k-class | 41.8 |
| 128k | 128k-class | 44.4 |
| 256k | **232k** | **48.0** |

The 256k cell is labeled by what landed, not what I aimed at. Likely mechanism: the hybrid sparse +
linear attention architecture on the card's spec. The curve is my measurement; the mechanism is not
isolated.

A real job at 128k of context held 54.3 structured, 51.4 code, 45.0 prose, tools parsing throughout.
The pool behind it: 12 GB KV per box, **1,145,238 token slots** — 2.86 full-window requests, or 16
seats at ~71.6k average context. Left it serving at depth overnight: streams clean by morning, pool
intact, decode still above 40 tok/s with 200k tokens of context live.

## What I did not prove

1. **A BF16 A/B.** Stated above; the hardware cannot make one.
2. **The mechanism behind the rising depth curve.** Card-spec attribution, not an isolated ablation.
3. **Standard evals.** GPQA and ToolEvalBench are NVIDIA's runs, two benchmarks, 198 questions on
   GPQA — a thin table. My receipts are fleet-level, not benchmark replications.
4. **Vision lanes.** The checkpoint is natively multimodal; I ran text only.

## Isolates

Nights of 27–28 Oct, endpoint `:8008`: night-1 empty-KV lanes (structured / prose / code at
1–2–4 streams), the concurrency sweep, ctx-decode depth ladder, needle battery, tool-call probes,
the 1267-token maths run, and the overnight serve-at-depth check. Three clocks — night 1, the
sweep, ctx-decode — never mixed in one column.

## Credit

Recipe, container and cluster wiring: [vr8vr8](https://github.com/myllmbox) (myllmbox on HF,
bilikaz on GitHub). Checkpoint and the QAD quality table: NVIDIA's
[GLM-5.3-Flash-FP4-Local](https://huggingface.co/nvidia/GLM-5.3-Flash-FP4-Local), base model ZAI
(MIT). QAD method: "Quantization-Aware Distillation for NVFP4 Inference Accuracy Recovery,"
[arXiv:2601.20088](https://arxiv.org/abs/2601.20088). The numbers are mine.
