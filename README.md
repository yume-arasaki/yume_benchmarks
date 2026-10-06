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

Next time I run something, it goes in `reports/` with a date.

Quick take from the newest write-up: the myllmbox gauntlet, run on my desks across three engines. Short French coding prompts (Mia's own check, 48 requests): TensorFold `q4` cuts 4, `fp8` cuts 1 (P 0.82), vLLM on identical weights cuts 1 with a better tail (0.86 mean, 0.57 worst), and myllmbox's Qwen dual cuts **0** (P 0.99, worst task 0.95). The gauntlet itself fails on model code in both engines — a CSS transform overriding an SVG placement, which I confirmed by eye and then reproduced under vLLM — so TensorFold is not the culprit there. The checkpoint is not one quantization either: 145.6 GiB of experts at EXL3 4-bit, 18.0 GiB of everything else left at BF16, and it is the serving config that compresses that 18 GiB. Shipped defaults are an aggressive choice, documented as such, and one flag away from the exact weights.

License MIT. The numbers are mine. The serve recipe is Mia's.