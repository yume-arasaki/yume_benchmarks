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
| 2026-10-14 | [DeepSeek V4.1 Flash — sudoingX dual-Spark](reports/2026-10-14-dsv41-sudoingx-dual-spark.md) | EXL3 2.7bpw + own kernels + DSpark k5, TP2, :8007 | 135.5 structured · flat 57.6–61.5 to 240k · towers 108.3/88.2 wall-bound | §21 v2+wall+mixture protocol, five artifact catches |
| 14 Oct 2026 | [GPU price ledger, US market 2026 — receipts for the price post](reports/2026-10-14-gpu-price-ledger.md) |
| 14 Oct 2026 | [RED-SNOW one-Spark Part 2 speed run (streaming protocol)](reports/2026-10-14-redsnow-part2-speed-run.md) — corrected-protocol grid: decode flat 50.1→44.4 across 28x ctx, TTFT 14→280 s, prefill 446-642 tok/s, pin A/B 22.9 vs 17.1, energy rail | `2026-10-12_8012_*` streaming isolates |
| 16 Oct 2026 | [GLM-5.3-Flash abliterated — pushing the 1M window, two DGX Sparks](reports/2026-10-16-glm53-flash-abliterated-1m-two-sparks.md) — the raise past 1M refused by the checkpoint; two grids (A/B), full ladder 1→128; needle 36/36; a **1,038,602-token prompt** answered in 944 s | decode 49.0→37.2 to 874k, prefill 1,554→1,130, TTFT 5.2→773 s | `2026-10-16_8008_[ctxb_]*` isolates |
| 17 Oct 2026 | [RED-SNOW-5.3-FLASH abliterated on two DGX Sparks — justin's TensorFold kit, verified](reports/2026-10-17-redsnow-dual-spark-justincasey-kit.md) — his 39.57 reproduced at 39.45 drafted; smoke 22/22, decode 31/31; the queue confirmed to 128 streams; 1M on the q4 latent with a 1,039,849-token prompt answered | wall 22.6→19.5 flat · TTFT 5 s→16 min · 662 MiB free at the window | `2026-10-17_8012[_ctx1m]_*` isolates |
| 21 Oct 2026 | [DeepSeek-V4.1-Flash-1017 on two DGX Sparks — Mia's TensorFold Zig kit, and a pin that no longer exists](reports/2026-10-21-dsv41-1017-tensorfold-two-sparks.md) — her C1 reproduced within 2%; her own `bench.py` at 58.2 / 119.2 against her 55–61 / 113–117 bands; the **1,048,576 window serves a 1,039,888-token prompt** in 669.9 s; the maths clock passes with thinking off *and* on; one reproducible `fence` at `code@32k` | peak 119.0 tok/s at C4 · 128/128 at C128 · prefill 2,004 → 1,590 tok/s to 882k | `2026-10-21_8007[_ctx1m]_*` isolates |

| 22 Oct 2026 | [Bench agent: local vs cloud on one desk](reports/2026-10-22-bench-agent-local-vs-cloud.md) |
| 26 Oct 2026 | [GPU rental ledger, global market 2026 — receipts for the rental post](reports/2026-10-26-gpu-rental-ledger.md) |

Next time I run something, it goes in `reports/` with a date.

Quick take from the newest write-up: Mia shipped a new checkpoint and a different engine, and I ran both. Her C1 numbers reproduce within 2% on my clocks, and when I ran her own `bench.py` the gap at four streams closed too — 58.2 tok/s at one stream against her 55–61, 119.2 at four against her 113–117. The grid peaks at a batch of four (119 tok/s, 1.00 J/token) and then queues to C128 with all 128 answered. This is also the first lane on this pair to pass my maths clock without a scratchpad, on both thinking pins. The window is the headline: 1,048,576 loads on a head that already runs a dashboard and a container set, and a 1,039,888-token prompt came back with three planted phrases in order after eleven minutes of prefill. The engine's own line says `0.00 GiB each` at that window, so treat it as one long conversation, not four. The install is the other half: the pinned checkpoint commit was force-squashed off the Hub and never carried a tokenizer, the head has to export NFS with `no_root_squash`, and `hf` has to be on the non-interactive PATH. Anyone starting from her README today hits all three.

License MIT. The numbers are mine. The kits are Mia's and justincasey's.
