# RED-SNOW-5.3-FLASH one-Spark — Part 2 speed run (streaming protocol, 2026-10-12/14)

Companion to `2026-10-12-redsnow-one-spark-receipts.md` (Part 1 + the correction trail). This report is the corrected-protocol speed grid: every decode number is streaming, first content token to last content token, TTFT reported separately. Wall-based numbers from the night run are retired and marked where they survive.

## Strain

`redsnow-sage-exl3/tabby-mixedk/vcruz/2.49bpw` — vcruz305 SAGE EXL3 2.49bpw (15 shards, 97.4 GiB) of RED-SNOW-5.3-FLASH (Blackfrost-AI abliterated GLM-5.3-Flash, 321B). Fork pin `7c1636f` (benched; merged tip `764b4ad` reads 25% slower shallow — pin A/B below), TabbyAPI `7a524a9`, MTP draft on, Q4 KV, 262k ctx, MAX_BATCH_SIZE=1. Serve spark-1:8012, single node.

## Protocol

- decode = usage.completion_tokens / (t_last_content_chunk − t_first_content_chunk), streaming, temperature 0
- TTFT = t_first_content_chunk − request_send
- composition probe at 8k/65k/240k: zero reasoning_content chunks, t_first_content == t_first_any (persona template is answer-first; decode windows are pure output)
- prefill_tok_s_implied = prompt_tokens / TTFT
- reps: grid lanes 2 each, depth ladder 1 per rung (9 rungs), agent lane 3

## Grid (isolate `2026-10-12_8012_grid-stream-4lane`)

| lane | decode peak | runs |
|---|---|---|
| structured (count 1-200) | 45.2 | 43.0 / 45.2 |
| code (clamp + doctest) | 35.9 | 32.7 / 35.9 |
| json (GPU catalog) | 45.7 | 45.6 / 45.7 (561 tok) |
| prose (400 words) | 27.5 | 24.6 / 27.5 |

TTFT 1.2-1.7 s on ~220-token prompts. Structured/JSON lead (speculation has predictable output to chase); prose is the floor.

## Depth ladder (isolate `2026-10-12_8012_ctx-stream-ttft-decode`)

| target | pt | TTFT s | decode tok/s | prefill tok/s |
|---|---|---|---|---|
| 8k | 6,281 | 14.1 | 50.1 | 446 |
| 16k | 12,364 | 22.2 | 48.6 | 558 |
| 32k | 24,533 | 30.6 | 38.1 | 630 |
| 49k | 36,701 | 41.4 | 35.7 | 627 |
| 65k | 48,869 | 59.7 | 46.6* | 627 |
| 98k | 73,205 | 78.6 | 46.2* | 649 |
| 131k | 97,541 | 152.3 | 46.7 | 640 |
| 196k | 146,213 | 227.9 | 45.3 | 642 |
| 240k | 178,465 | 280.1 | 44.4 | 637 |

(*) 65k/98k cells from the dense-ladder re-run at those depths; see isolate rows for per-rung source.

Decode decline: **-11.4% across 28x context** (50.1 → 44.4). Comparators same-family: CYBER-FROST qwen38 lane -15.6% (47.3→39.9, `2026-10-07`), aligned GLM-5.3-Flash TensorFold TP2 -7.5% (54.9→50.8, `2026-10-02`). Flat is the finding.

TTFT is the depth cost: prefill holds 446-642 tok/s, so wait ≈ ctx/550 seconds.

## Agent lane (isolate `2026-10-12_8012_agent-lane-rerun-3rep`)

Research-app turn, 32,913 pt, tools attached: decode 33.6 / 32.0 / 31.9, TTFT 55-57 s. No tool call fired in any rep (model answers directly, 66-105 tok). Supersedes the retired 2026-10-09 lane (78.1 over a 154-tok/1.97 s window — artifact class, retired).

OMP tower (wall-based night isolate `2026-10-09_8012_omp-workload-35k`): 16.4 tok/s wall, 2,048-tok generate, 31,600 pt. Prefill folded in; labeled wall on-panel, never cross-compared with streaming cells.

## Pin A/B (serve boot self-tests, `~/rs-night/serve.log` + `pin-restore.log` on spark-1)

- `7c1636f`: 678 tok in 22.9 s, draft 415/610 accepted (68%)
- `764b4ad` (auto-enables glm5_next recurrent path): 479 tok in 17.1 s, draft 288/447 (64%)
- same weights, same serve config, two-point; -25% on the merged tip, no error logged. Pin your commits.

## Energy / cost rail (structured C1 basis, 45.2 tok/s streaming)

- 1.29 J/tok — night structured C1 isolate energy block (1,622.7 J / 1,260 tok)
- depth curve (wall-based night ladder, prefill-dominated — labeled): 6.3 J/tok @ 8k → 114.2 J/tok @ 240k; rail 68.8 → 75.3 W
- local $0.65/day = 45.2 × 86,400 × 1.29 J → kWh × $0.1834 (EIA), duty 1.0
- ceiling 3.91 M tok/day = 45.2 × 86,400
- GROK-4.6 API $23.45/day = same tokens × $6.00/M output (price key 2026-10-06)

## Concurrency (isolate `2026-10-12_8012_conc-sweep-c1c4`)

Serialized serve (MAX_BATCH_SIZE=1): sum-of-streams structured 37.2 / 50.3 / 69.9 at C1/C2/C4, wall aggregate flat 36.5-37.2 — queue, not parallelism. Printed, never charted as throughput scaling.

## Voided / retired (kept for the trail)

- `2026-10-12_8012_depth-falloff-investigation` + `depth-fixtest-161` — VOID: wall-based probe, prefill folded into decode; the "O(N) decode tax" was TTFT growth. 1.5.1 vs 1.6.1 rebuild comparison stands only as wall-no-difference.
- `2026-10-09_8012_hermes-workload-35k` 78.1 — RETIRED (short-window artifact).
- Night grid lanes (43.2/40.7/32.1/29.2) — superseded by the streaming set above.

## Companion dash

`benchmarks/design/redsnow-grid-dash/` (thirteenth agent-grid strain). 27 animated IDs + 4 bar heights readback-verified against these isolates; render 2.1 MB / 540 frames.
