# Mac lane receipts — Qwen3.8-Flash-Next on M4/M5, oMLX vs TensorFold (2026-10-08)

Post basis: `drafts/2026-10-08-mac-lane-flashnext-post.md` (research doc: `2026-10-08-mac-lane-flashnext-research.md`). Every printed number → source line. Labels: PROJECT (engine's own DB/notes), REPORTED (community), FIRST-PARTY (ours), SPEC.

## Chip rows

- M4 Pro 64 GB, oMLX oQ4e-mtp, offload 0.45: 10.9 @ 4k / 9.7 @ 64k / 9.9 @ 128k — PROJECT, omlx.ai/benchmarks/performance/kgjdad7z
- M4 Max 128 GB, oMLX oQ4e-mtp: 62.0 @ 1k / 67.6 @ 4k; batching 2× 145.2 / 4× 224.1 — PROJECT, omlx.ai/benchmarks/performance/t02sr2s4 (also gtglrzrx 69.1 @ 1k, zcwb5aod 65.5, 137jwnyg 41.0 older)
- M5 Pro: ~60 t/s gen / ~900 PP — REPORTED, mlx-serve Sushi-4 user aside via llamaperf
- M5 Max 128 GB base oQ4e: 59-66 across Oct 1-5 DB rows (59.0/46.9 tc3yfm0q, 60.9 2hw0b95j, 64.5 0in9qlee, 58.1 @ 128k lb9z1zgy) — PROJECT
- M5 Max MTP-on Uncensored oQ4e-100K-MTP: 62.0 @ 1k / 80.5 @ 8k / 66.0 @ 128k, PP 1,853-2,450 — PROJECT, 2cn5nfyy
- M5 Max mlx-serve Sushi-4: 90 t/s / 1,900 PP — REPORTED, llamaperf M5 Max page (Sep 29)
- M5 Max ds4 fork sf-q3-8flash Q2+MTP: 86.7 (stock 75.8; Q4 77.8→85.9), bit-exact — REPORTED, llamaperf (Oct 8)
- M5 Max MTPLX (Youssofal pack): 73.5 w/ MTP vs 43.8 plain, PP 1,453 @ 4k / 1,094 @ 65k, 262k ctx, 87 GiB peak — REPORTED, r/LocalLLM "Qwen 3.8 Flash-Next MTPLX is a beast" (sentinel 1w3g6rx) + llamaperf row
- M5 Max llama.cpp UD-IQ4_XS agent run: 24-36 sustained, ~1,000 PP, 75 GB — REPORTED, r/LocalLLaMA (sentinel 1vzg7fl)
- M3 Ultra 256, TensorFold: 105-107 homepage; 111.1-142.3 release notes 0.3.4.1 — PROJECT, tensorfold.dev + notes via vramcalculator.com/tensorfold
- M5 Ultra 64c 256, oMLX base: 96.7 @ 4k (4qvcf5qu), 98.6 @ 64k (1uf1dlpq), PP 4,685-4,827 — PROJECT
- M5 Ultra 80c 256, oMLX TQ-KV4 + Lightning MTP: 157.7 @ 4k, 158.9 @ 32k, 96.6 @ 195k, PP 5,081-5,663, 101 GB peak — PROJECT, omlx.ai/benchmarks/performance/a3tzjs1s
- M5 Ultra 256, MTPLX: 58 @ 4k — REPORTED, llamaperf M5 Ultra page
- 2× M5 Ultra TP=2 TensorFold: 114 t/s prose / 1,600 PP / TTFT 0.06s = GLM-5.3-Flash, NOT Qwen — fxtwitter ashxhart/status/2107921303894151584 (printed as the misread-warning only)

## M4→M5 TTFT/accelerator beat

- M4 Max PP 558-682 vs M5 Max PP 963-2,508, same 40c GPU — PROJECT (rows above); bandwidth 546→614 GB/s SPEC (memorybandwidth.dev + Reddit German thread)
- M5 per-GPU-core neural accelerators — SPEC, Apple newsroom Oct 2025 (apple.com/ca/newsroom/2025/10/apple-unleashes-m5/), MLX research post Nov 2025 (machinelearning.apple.com/research/exploring-llms-mlx-m5), dwijen.com kernel writeup
- TTFT: M5 Max 4.3s @ 4k vs M4 Max 6.8-7.4s @ 4k — PROJECT (DB rows); 56.3s @ 128k M5 Max — PROJECT lb9z1zgy
- oMLX ANE prefill +13% @ 16K / +57% @ 32K on M3 Max (Qwen3.8-27B) — PROJECT, github.com/jundot/omlx issue #2781
- M5 Max thermal -30% after ~1 min — REPORTED, kernel-dev comment in sentinel 1vyfved snapshot

## Engine comparison

- plotarmordev M5 Ultra head-to-head (NOT weight-matched): TF 4-bit 4.26 GB/tok vs oMLX 5-bit 5.34; TF +15% drafted, 353 vs 226 @ 8 streams, 109 vs 124 GB; oMLX prefill 1.8×, +7% undrafted — REPORTED, via vramcalculator.com/tensorfold
- WescheNex1q M4 Max three-engine (27B): TF 154 / oMLX 146 / mlx-serve 146 code cells, 29-31 all undrafted — REPORTED, same
- TF 0.6.2 "Flash Next faster on Macs at 64K to 128K" — PROJECT, release notes
- TF Flash-Next Mac figures flagged "4-bit models, short thinking replies, earlier builds. Current-release benchmarks are pending" — PROJECT, tensorfold.dev

## First-party anchor

- 1× DGX Spark, cousin qfn 523ecd3 (EXL3 3.05 bpw): 102.3-107.7 lanes at 273 GB/s, ~2.29 GB/token active — FIRST-PARTY, runs/2026-09-20_8009n_* (bytes-touched teach)
- cyberfrost SAGE 4.0 bpw same box 85.1 — FIRST-PARTY, runs/2026-10-07_8012_*

## Cut/downgraded (recorded)

- "100 tok/s M5 Max Reddit" base-model claim: NOT FOUND; closest verified = heretic-2 FINE-TUNE 102.4 @ 1k on M5 Ultra 96 GB (3f9a3j0n) + burst-thermal candidates. Post prints 73.5/86.7/90 as the real M5 Max ladder with the forensic note.
- LLMCheck 84-85 M5 Ultra / 44-45 M4 Max: echo-grade, cut.
- oMLX 28.6 t/s Thunderbolt: Qwen3.6-27B, different family, cut (known trap).
- dev.to A100 97 t/s: not Mac, cut.
- baguaai 2-bit YaRN stress test: writeup self-contradicts (35.8k vs 350K), unresolvable from snapshot, cut pending OP.
- M4 Ultra: does not exist (Ars Technica/macrumors Mar 2025; Apple Studio M5 newsroom Aug 2026). Structural note, not a row.

## Link verification

- HF packs: Jundot/Qwen3.8-Flash-Next-oQ4e-mtp, TensorFold/Qwen3.8-Flash-Next-MLX-oQ4-MTP verified live 2026-10-06 (prior ledger); Youssofal/Qwen3.8-Flash-Next-MTPLX-Optimized-Speed to re-verify via HF API before reply ships.
- oMLX DB URLs: fetched this session.
- tensorfold.dev, vramcalculator.com/tensorfold: fetched this session.
