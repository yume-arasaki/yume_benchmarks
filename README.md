# yume_benchmarks

What I measured on my desks. [@yume_arasaki](https://x.com/yume_arasaki)

Not a tool. Not a recipe. Just the write-ups.

| Date | |
|---|---|
| 9 Sep 2026 | [GLM-5.3-Flash EXL3 on two DGX Sparks](reports/2026-09-09-glm53-flash-exl3-two-sparks.md) |
| 14 Sep 2026 | [DeepSeek V4.1 Flash EXL3 on two DGX Sparks](reports/2026-09-14-dsv41-flash-exl3-two-sparks.md) |
| 15 Sep 2026 | [RTX 4090 drag race on Qwen 3.8 27B](reports/2026-09-15-4090-drag-race-qwen38-27b.md) |
| 17 Sep 2026 | [RTX 4090 engine swap: EXL3 + DFlash2 on Qwen 3.8 27B](reports/2026-09-17-4090-exl3-dflash2-qwen38-27b.md) |
| 18 Sep 2026 | [Qwen3.8-Flash-Next NVFP4 on two DGX Sparks](reports/2026-09-18-qwen38-flash-next-nvfp4-two-sparks.md) |
| 19 Sep 2026 | [Qwen3.8-Flash-Next EXL3 on one DGX Spark (`grid_test_failed`)](reports/2026-09-19-qwen38-flash-next-exl3-one-spark.md) |
| 19 Sep 2026 | [Qwen3.8-Flash-Next EXL3 native on one DGX Spark](reports/2026-09-19-qwen38-flash-next-exl3-native-one-spark.md) |
| 20 Sep 2026 | [Qwen3.8-Flash-Next EXL3 native on one DGX Spark (recipe speed)](reports/2026-09-20-qwen38-flash-next-exl3-native-one-spark.md) |
| 23 Sep 2026 | [MiMo-V2.6-Flash-RL on two DGX Sparks (day-zero recipe)](reports/2026-09-23-mimo-v26-flash-two-sparks.md) |
| 26 Sep 2026 | [Qwen3.8-Flash-Next hibrid48 on two DGX Sparks (full agent grid, YaRN 1M)](reports/2026-09-26-qwen38-flash-next-hibrid48-two-sparks.md) |
| 28 Sep 2026 | [Qwen3.8-Flash-Next hibrid48 v4.1 on two DGX Sparks (second grid, Hermes PR)](reports/2026-09-28-qwen38-flash-next-hibrid48-v41-two-sparks.md) |
| 1 Oct 2026 | [GLM-5.3-Flash TensorFold on two DGX Sparks (engine swap, full grid)](reports/2026-10-01-glm53-flash-tensorfold-two-sparks.md) |
| 2 Oct 2026 | [GLM-5.3-Flash on Mia's own TensorFold recipe, two DGX Sparks (aligned weights, flat falloff)](reports/2026-10-02-glm53-flash-mia-tensorfold-two-sparks.md) |
| 3 Oct 2026 | [Qwen3.8-Flash-Next on one DGX Spark (bilikaz v5.1, full grid)](reports/2026-10-03-qwen38-flash-next-one-spark.md) |
| 6 Oct 2026 | [Qwen 3.8 Flash map post — receipts ledger](reports/2026-10-06-qwen38-flash-map-receipts.md) |
| 6 Oct 2026 | [TensorFold (Mia) vs vLLM (Mia) vs myllmbox — same weights, three engines, one gauntlet](reports/2026-10-06-tensorfold-vs-vllm-vs-myllmbox.md) |
| 12 Oct 2026 | [RED-SNOW-5.3-FLASH abliterated on one DGX Spark — receipts ledger](reports/2026-10-12-redsnow-one-spark-receipts.md) |
| 14 Oct 2026 | [RED-SNOW one-Spark Part 2 speed run (streaming protocol)](reports/2026-10-14-redsnow-part2-speed-run.md) — corrected-protocol grid: decode flat 50.1→44.4 across 28x ctx, TTFT 14→280 s, prefill 446-642 tok/s, pin A/B 22.9 vs 17.1, energy rail | `2026-10-12_8012_*` streaming isolates |
| 16 Oct 2026 | [GLM-5.3-Flash abliterated — pushing the 1M window, two DGX Sparks](reports/2026-10-16-glm53-flash-abliterated-1m-two-sparks.md) — the raise past 1M refused by the checkpoint; two grids (A/B), full ladder 1→128; needle 36/36; a **1,038,602-token prompt** answered in 944 s | decode 49.0→37.2 to 874k, prefill 1,554→1,130, TTFT 5.2→773 s | `2026-10-16_8008_[ctxb_]*` isolates |

Next time I run something, it goes in `reports/` with a date.

Quick take from the newest write-up: the abliterated GLM-5.3 lane at its ceiling. The recipe already boots at 1,048,576, so asking for more gets `tensorfold: --context 2097152 exceeds this model's 1048576-token window` — the checkpoint is the wall, not the box. I re-gridded the lane twice (two boots, 13 isolates each, full ladder 1→128) and then drove the top of the window directly: a **1,038,602-token prompt** with phrases planted at 5/50/95%, all three returned in order, 944 seconds. Needle retrieval across the ladder is **36/36** — three placements per depth now, not three repeats of one. Decode falls 24% to 874k and prefill to 1,130 tok/s with 773 s to first token, so the top of this window is batch work, not a chat; the head node also finishes a full-window prefill with 0.4 GiB of the kit's own 8 GiB memory floor left. The row that made me rewrite a claim: the OMP 35k tower failed in one boot and came back clean in the other, on identical prompts, so the clean tool-calling result I published for this tower in October is not stable.

License MIT. The numbers are mine. The serve recipe is jayleaton's.
| 2026-10-14 | [DeepSeek V4.1 Flash — sudoingX dual-Spark](reports/2026-10-14-dsv41-sudoingx-dual-spark.md) | EXL3 2.7bpw + own kernels + DSpark k5, TP2, :8007 | 135.5 structured · flat 57.6–61.5 to 240k · towers 108.3/88.2 wall-bound | §21 v2+wall+mixture protocol, five artifact catches |
| 14 Oct 2026 | [GPU price ledger, US market 2026 — receipts for the price post](reports/2026-10-14-gpu-price-ledger.md) |
