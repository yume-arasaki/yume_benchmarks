# Inkling 2 Mini on maya-ik2 — RTX 4090 agent bench grid

Date: 2026-11-24 (bench nights 2026-11-23/24, +07)
Strain: `inkling-2-mini-ud-q4kxl/maya-ik2-43884ff/masterpig2-rtx4090`
Golden: sirius `art_1a20994930b541ca4` (operator-elected)

Engine: [alesha-pro/maya-ik2](https://github.com/alesha-pro/maya-inkling-mini) by Alexey
Fateev (@superalesha), on Project Maya, commit `43884ff`, built for sm_89 on CUDA 12.4.
Model: Thinking Machines' Inkling 2 Mini, 26B looped MoE (24 layers, loop 6-17 ×3,
48 layer visits, 3,072 experts). Weights: Unsloth `UD-Q4_K_XL` GGUF, FP8 KV, official
DSpark drafter. Hardware: 1x RTX 4090 (450 W limit), headless during all runs,
344,064-token context (262,144 with the drafter).

Quality gate before any speed claim: 90/96 GSM8K, greedy, through the engine's server.

## Decode ladder — engine CLI, his documented protocol

Cold start every row, 3 reps, medians. Plain arm ctx 344,064; drafter arm ctx 262,144,
`IK2_DRAFT_CTX=65536`. Env: `IK2_KV=fp8 IK2_GATE_TILE=64 IK2_GATE_BF16=1
IK2_Q8_UNROLL=1 IK2_BATCH_PREFILL=1`.

| Depth | Prefill tok/s | Plain decode | Drafter decode | Peak VRAM |
|---:|---:|---:|---:|---:|
| 4,096 | 5,275 | 151.1 | 228.6 | 23.7 GiB |
| 63,981 | 5,057 | 128.9 | 195.6 | 23.7 GiB |
| 127,910 | 4,503 | 112.2 | 189.8 | 23.7 GiB |
| 340,000 | 3,269 | 76.3 | n/a (262K cap) | 23.7 GiB |

Reference, printed never subtracted: the author's RTX 3090 (300 W, greedy, real
documents) decode 124.2 / 105.6 / 90.6 / 62.3 and prefill 2,742 / 2,361 / 2,079 / 1,473
([tweet](https://x.com/superalesha/status/2124102269138346409)). Our ladder runs on his
exact CLI command shape and env; prompts are random token ids and engine-default
sampling. Decode at depth is KV-read bound and content-independent; prefill is not.

His card through the server peaked at 182-207 tok/s on code; our CLI drafter: 228.6.

## The VRAM residency trap (measured both ways)

The engine needs 22.7 of 24 GiB for every expert in VRAM. With a desktop session
holding ~1 GiB it silently spills experts to pinned host RAM (3.8 GiB/s over PCIe):
decode settles at 91.7 / 76.1 / 64.7 / 44.6 across the same depths, a flat 0.72x at
every depth, while prefill barely notices. Headless boot restores the table above.
Read the engine's startup line (`resident: every expert in VRAM`) before trusting
counters on this model. Night-1 dataset kept as `ladder-night1.jsonl`.

## Frozen agent grid — server cells, T=0, thinking off

- sparkdash lanes (C1 decode): structured 156.6, prose 156.1, code 156.8, json 159.9
- energy: 1.0-1.4 J/token, single-4090 rail (nvidia-smi power.draw, trapz)
- needle-5-50-95-v1: **12/12** exact code hits, deepest row 288,018 measured prompt
  tokens (3/3 at 9k / 36k / 144k / 287k)
- ctx-decode-256-noeos-v1 (this build rejects `ignore_eos`; new protocol id per the
  new-clock rule): decode 148.7 → 84.7 from 9,075 → 287,394 actual tokens, prefill
  5,398 → 3,634
- job-at-depth-v1: structured/prose/code jobs hold 141.8 → 105.3 tok/s from 32k → 144k;
  **tools call at 32k, answer-in-text at 128k** (recorded as a fail row)
- omp-tower-35k-v1: full 2,048-token single-file build at 94.7 tok/s, finish length
- hermes-research-app-35k-v1: two real web_search calls (FSRS queries), 8.2 s wall,
  36,994 prompt tokens
- concurrency: the server queues streams instead of erroring (C2 probe, both complete)

Isolates: `runs/2026-11-24_18000_*` (13 files) in the private ledger; CLI ladder
`ladder.jsonl` on the bench host. Derived economics: 13.49 M tok/day single-stream
(prose C1 × 86,400); $81/day at Grok-4.6 list ($6/M output).

## Conclusion

An engine written for a 3090, run two days after release, beats the 3090 at every cell
on a four-year-old 4090: 1.22-1.24x decode at all depths, up to 2.22x prefill, perfect
retrieval to 288K, and a joule count per cell. The decode kernels remain Ampere-tuned;
the drafter path is where Ada caught up first. The 3090 numbers are the author's, from
his tweet, on his harness.
