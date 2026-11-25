# Inkling 2 Mini on maya-ik2 — 300K fidelity arm vs the 1017 gold standard

Date: 2026-11-25 (bench 2026-11-24/25, +07)
Strain: `inkling-2-mini-ud-q4kxl/maya-ik2-43884ff/masterpig2-rtx4090`
Instrument: drift-study-v2-300k, frozen battery `5dff9ead0573` (168 items), thinking
pinned off, T=0, one sample per cell, static filler, single-tenant headless 4090.

Engine: [alesha-pro/maya-ik2](https://github.com/alesha-pro/maya-inkling-mini) @43884ff
(sm_89, CUDA 12.4), Unsloth UD-Q4_K_XL, FP8 KV, 344,064 ctx. Companion data: the
agent grid report (2026-11-24) and the [DSV41-1017 two-arm fidelity suite]
(../2026-11-21-dsv41-flash-two-arm-fidelity-suite.md) whose battery and protocols this
arm reuses. Reference receipts below are the suite's dual-Spark arm
(`dsv41-1017-flash-exl3/tensorfold-tp2/mia/c7f2b85`) unless labeled cloud.

## What we ran

- CHALLENGER: Inkling 2 Mini (26B looped MoE) on 1× RTX 4090, maya-ik2 @43884ff,
  UD-Q4_K_XL, FP8 KV, headless, single-tenant.
- GOLD STANDARD: DeepSeek V4.1 Flash 1017, EXL3 EAQ v1 2.9 bpw, two DGX Sparks, TP=2
  (the 2026-11-21 suite's local arm; cloud BF16 control cited where labeled).
- EXAM: the suite's frozen 168 questions — arithmetic 54, extraction 40, conditional 20,
  tool_call 36, rule_bound 18. T=0, one answer each, thinking off unless stated.
- DRIFT LADDER: exam re-taken as one conversation grows to 294,494 tokens.
- TOOL CHAIN: 43 calls, two planted failures, run fresh and at depth.

## Fresh-context battery, per pin (never one column)

| arm | reasoning | score/168 |
|---|---|---:|
| GLM-5.3-Flash (battery author, calibration ceiling) | off | 164 |
| DeepSeek cloud, provider BF16 | off | 96 |
| DeepSeek cloud, provider BF16 | on | 145 |
| DeepSeek 1017, 2× DGX Sparks | off | 95 |
| DeepSeek 1017, 2× DGX Sparks | on | 142 |
| Inkling 2 Mini, 1× RTX 4090 | off | 78 |
| Inkling 2 Mini, 1× RTX 4090 | on | 90 |

Distance to the Spark rig: 17 points off-pin, 52 on-pin. The Spark arm sits 1-3 points
behind its own cloud control — that is the suite's quant-fidelity receipt, not noise.
Instrument caveat, stated up front: the battery was built and frozen on GLM-5.3-Flash
(waiver: calibration family at ceiling), and its grader reads the first number in the
response, string-compared to the key. Absolute cross-family gaps are therefore
GLM-shaped; the class-level and within-arm comparisons below carry the weight.

Class-level, same pins: arithmetic 0/54 both pins for Inkling vs 41/54 (Spark,
reasoning on) and 42/54 (cloud, reasoning on); rule_bound 0/18 off, 18/18 on;
extraction 40/40 off (28/40 on, 12 no-answer: reasoning consumes the 640-token budget).

## Ladder (alpha session, thinking off)

| rung | actual pt | score/168 |
|---|---:|---:|
| fresh | 97 | 78 |
| 50k | 49,157 | 73 |
| 100k | 103,695 | 73 |
| 150k | 147,329 | 74 |
| 200k | 190,975 | 72 |
| 260k | 250,956 | 73 |
| 300k | 294,494 | 74 |

One-time −5 step when context first lands; flat band 72-74 for the next 245K tokens.
Slope −0.0094 points/1K tokens. Bootstrap on fresh→300K (2,000 resamples, frozen seed):
Δ −0.024, 95% CI [−0.131, +0.083] — contains zero. On this battery, 294,494 tokens of
history is not measurable as degradation. (The stock `analyze` command requires 6-stage
ladders; this 7-rung isolate was analyzed with the same slope/CI math inline.)

Near-miss evolution: digit_off flat at 18 at every rung (pin/quant noise); wrong_branch
improves 6→3; wrong_tool_similar grows 7→19 (tool-name confusion, not reasoning loss).

## Retention vs the gold standard's ladder

- Inkling: 78 → 74 at 294,494 = **94.9%** retained (thinking off every rung)
- DeepSeek 1017, 2× Sparks: 95/96/91/94/94/93 at 84 → 915,246 = **97.9%** retained
  (complete static-filler ladder, 2026-11-20 isolate)
- DeepSeek cloud BF16: 98 → 92 at 919,677 = 93.9% (different arm, labeled)
- Needle (same night, Inkling): 12/12, deepest at 288,018

Both ladders flat; retention parity holds under every arm. The pinned run zeroes
arithmetic for DeepSeek and Inkling alike (0/54 both arms off-pin) — that is a pin
phenomenon, not a model deficit.

## Thinking-on battery (separate pin, never blended)

90/168 fresh. Class profile shifts, not uniformly up: rule_bound 0→18 (fully recovered
by reasoning), conditional 14→19, tool_call 24→25; extraction 40→28 with 12 no-answer
items; arithmetic remains 0/54. Recovery shape differs from DeepSeek, which regains
arithmetic (41-42/54) under the same pin.

## Tool chain (within-arm compare)

- fresh: **43/43**, both scripted failures (positions 12, 28) recovered on retry,
  zero branch failures
- at depth: **43/43** at 261,681 measured prompt tokens, 105.9 s wall, zero breaks
- Reference (same instrument, other arm): DSV41-1017 local 43/43 at 919,234

## Read it right

- One sample per cell; T=0 is not determinism.
- Battery calibrated on GLM-5.3-Flash, ceiling-waived, first-number grader: absolute
  cross-family scores are GLM-shaped; near-miss evolution and within-arm deltas are the
  sensitivity signals.
- Retention is the honest compare, printed per arm with its own depth. Never one column.
- One planted checksum canary came back wrong at 256,064 (expected 2988730, got
  954103), n=1, recorded not smoothed; the 12/12 needle is the counterweight.
- Isolates: `runs/2026-11-24_18000_drift-study-v2-alpha-300k.json` (+ .session.jsonl),
  `runs/2026-11-25_18000_tool-chain-40-v1.json`,
  `runs/2026-11-25_18000_tool-chain-at-depth-256064.json`; reference isolates
  `runs/2026-11-20_8007_drift-study-v2-alpha-900k.json` (Spark ladder),
  `runs/2026-11-19_8007_battery-v2-local{,-thinking}.json` and
  `runs/2026-11-19_api_battery-v2-cloud{,-thinking}.json` (per-pin cells),
  `runs/2026-11-24_18000_needle-5-50-95-v1.json`.
- Ops notes: one ladder kill at the 260K rung (transient tailnet drop, 4 fast retries
  exhausted); `--resume` restored from the 200K checkpoint at zero data loss. Occupancy
  recorded per stage, all single-tenant.
