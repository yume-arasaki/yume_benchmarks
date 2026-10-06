# Qwen 3.8 Flash map post — receipts ledger

Post: `2026-10-06` map ("Insane developments on Qwen 3.8 Flash"). Every row in the post maps to a source here. `FIRST-PARTY` = our JSON on disk, `MEASURED` = community receipt we verified live, `CLAIM` = unverified/estimate — the post only prints FIRST-PARTY and MEASURED.

## Benchmarks vs Opus 4.8

- Artificial Analysis live comparison page (fetched 2026-10-06): index 40 vs 42; Terminal-Bench 4.0 25% vs 22%; AutomationBench-AA 56% vs 46%; GDPval-AA 1612 vs 1438; HLE 38% vs 49%; blended price $0.0882 vs $3.85 per 1M.
  `https://artificialanalysis.ai/models/qwen-3-8-flash-next` (compare vs Opus 4.8)
- SWE-bench Pro 69.2 (Opus) = Anthropic Opus 4.8 System Card §8.2 (`https://www.anthropic.com` → system card PDF, mirrored in workspace `dd_cache/opus48.html` + cached system-card markdown). 62.5 (Flash-Next) = Qwen model card, Claude Code harness, refined benchmark. Cross-vendor self-reports, different harnesses — the post labels this.
- Architecture (512 experts, 10 routed + 1 shared, 125B + 51B n-gram + 4B MTP, 262k native): HF card `Qwen/Qwen3.8-Flash-Next`.

## Lane C — Strata engine

- Strata repo: `https://github.com/Niko1221/Strata` — MIT, model-family-specific, v0.1.39 out 2026-10-04. README reference rig: RTX 5070 12 GB + Ryzen 5 7600 + 64 GB, Q2_0, ~94 t/s write / ~2,650 t/s read @ 32K, spec floor 12 GB VRAM / 32 GB RAM, ~70 GB download (plan ~80 GB disk). Engine 0.1.36 footnote on the rig numbers.
- EpicMaan receipt: `https://x.com/EpicMaan/status/2107133815441555898` — 5090, GSQ-RCO IQ3_S, 160 t/s, "was 15 t/s on llama.cpp." Verified 2026-10-06 via fxtwitter. Engine is Strata-by-thread-context (tweet is a reply into a Strata promo thread, never names it) — post says "in a Strata thread."
- 5000e12 receipt: `https://x.com/5000e12/status/2107060996909240817` — 4090, Strata, IQ3 Corder, ~60 decode / ~2,500 prefill. JP. Verified 2026-10-06.
- danielbitpro 5070 "90 t/s": `https://x.com/danielbitpro/status/2107220374375072141` — CUT from the post. 37 views, no methodology, number matches no measured README row — README paraphrase with drifted number. Kept here as a cut receipt.
- MinLiBuilds (CN): `https://x.com/MinLiBuilds/status/2107133331012026371` — five-round VRAM-budget sweep, modded 48 GB 4090, "显存依然很重要" (VRAM still matters). Not in post body; teaching material.
- RoundtableSpace promo thread (the "your gaming PC can run 125B" thread EpicMaan replied into): `https://x.com/RoundtableSpace/status/2106848080179909040`.

## Lane B — GGUF / llama.cpp

- eirrann_art receipt: `https://x.com/eirrann_art/status/2107233442476003353` — 5950X + 3090 + 64 GB, UD-Q2_K_XL, `n-cpu-moe=32`, 28.1 decode / 244.9 prefill @ 65k ctx, sustained 1.59M production tokens. Second window 33.7/255.2. Verified 2026-10-06.
- Unsloth UD-IQ1_S "78 GB RAM minimum": Unsloth documentation.
- r0b0tlab EXL3 pack: `https://huggingface.co/r0b0tlab/Qwen3.8-Flash-Next-ExL3-2.50bpw` — 41.5 GiB weights + 18.5 GiB n-gram NVMe-streamed; single 3090 24 GB (20.7 GB peak), ~59 GB host RAM; 38.6 t/s decode with MTP (27.8 without; 20.9 @ 175k ctx), 664 t/s prefill; NIAH pass at 262,080 (2- and 3-needle); Q200v2 mean 51.2 t/s over n=180. Verified verbatim 2026-10-06, live HF API + raw README.
- V100 e-waste lane: pentacoxian-dev quant `https://huggingface.co/pentacoxian-dev/Qwen3.8-Flash-Next-IQ3E-Q8D-MTP-GGUF` (96k downloads) + _ryu15_ receipt `https://x.com/_ryu15_/status/2103880612654490059` — dual V100 32 GB (64 GB total), ~60–88 t/s @ 262k ctx, Sep 26. Verified  entry + tweet 2026-10-06.

## Lane D — mobile

- Tono_Ken3 receipt: `https://x.com/Tono_Ken3/status/2105523551122104560` — RTX 4060 laptop 8 GB, Strata + Q3, 30 t/s, `-c 192K`, 130 W (JP text: server + MacBook Air client combined). Verified 2026-10-05/06.

## Quants

- ISTA-DASLab GSQ-RCO: `https://huggingface.co/ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF` (raw README re-fetched 2026-10-06, snapshot in workspace `dd_cache/ista_readme_raw.md`): IQ3_S 3.50 bpw, 54.8 + 28.8 GB shards = 83.6 GB total; task avg 93.26 vs BF16 base 93.12 (AIME25 100.00, GPQA-D 92.93 vs 91.92, LCBv6 86.86) — above BF16 base on their suite.

## Mac lane (64 GB+ unified)

- Engines (two, not just GGUF): TensorFold on M1-M4 (oQ kernels, scaled n-gram table, dense projections on matrix units pre-M5; releases v0.4.0-v0.6.3) and oMLX (owns the oQ format, SSD-paged tiered KV, menu-bar app).
- Official TF Mac checkpoint: `https://huggingface.co/TensorFold/Qwen3.8-Flash-Next-MLX-oQ4-MTP` (oQ2 variant also published). Live via HF API 2026-10-06.
- oMLX author's checkpoint: `https://huggingface.co/Jundot/Qwen3.8-Flash-Next-oQ4e-mtp`. Live via HF API 2026-10-06.
- oMLX serves Flash-Next: evidenced by TF v0.3.6.3 benching its prompts "level with oMLX at 32k and 64k" on M3 Ultra.
- Engines: `https://github.com/ashhart/TensorFold` · `https://github.com/jundot/omlx`.
- HONEST GAP: no published absolute t/s for the 180B on any Mac, any engine. Sub-64 GB Mac lane = Qwen3.8-27B dense (131-160 t/s M5 Ultra DFlash2 oQ4e, TF v0.6.0; 16 streams @ 32k on 64 GB, TF v0.4.0). Trap: oMLX's 28.6 t/s Thunderbolt bench is Qwen3.6-27B, different family.

## First-party rows (repo reports)

- 1x Spark EXL3 native (107 count / 71.7 prose / 15-15 needle @ 243k): `reports/2026-09-20-qwen38-flash-next-exl3-native-one-spark.md`
- 2x Spark TP2 NVFP4 (68 count): `reports/2026-09-18-qwen38-flash-next-nvfp4-two-sparks.md`
- 2x Spark hibrid48 v4.1 (138 count / 383 C4 / 200.5 prose / needle @ 900k): `reports/2026-09-26-qwen38-flash-next-hibrid48-two-sparks.md` + `reports/2026-09-28-qwen38-flash-next-hibrid48-v41-two-sparks.md`
- 27B honest-pointer report: `reports/2026-09-15-4090-drag-race-qwen38-27b.md`

## Mislabel traps (kept OUT of the post)

- "V100 32 GB @ 39.7 t/s" single-card — no such receipt; V100 work is dual-card genuine Flash-Next (above).
- "RX 7600 8 GB @ 30 t/s" → Qwen3.6-35B-A3B, different family (`https://x.com/Manz/status/2063967868853616797`, ~June 8).
- MTP speedup pair 83.2→138.8 t/s (1.67x) — unpinnable; llama.cpp PR #28243 closed, not merged, pair appears nowhere. MTP is rig-dependent (helps 5090-class + 3090-EXL3; regresses dual-3090 −10%, 8 GB sm_89 −30–50%,  5070 Ti 20.7→16.2).

## How-to pointer

- Strata one-click installer + `ANTHROPIC_BASE_URL=http://localhost:8080` — repo README quickstart.

---

*Post: workspace-research/drafts/2026-10-06-qwen38-flash-map-post.md. DD pass: deleg_c76407c8 (2026-10-06), evidence in workspace/dd_cache/.*
