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

Next time I run something, it goes in `reports/` with a date.

Quick take from that write-up: two Sparks, Mia's EXL3 recipe `9c0794b`, thinking off. Count-to-200 **66.4 / 133.4 / 187.6** at 1 / 2 / 4 streams (she posts 62.9). Essay **28.5 / 57.3 / 66.4**. Clamps **61.3 / 120.7 / 177.3**. Long context doesn't fall over. A warm session writing a tiny game is **43.8**. I'm not averaging those.

License MIT. The numbers are mine. The serve recipe is Mia's.