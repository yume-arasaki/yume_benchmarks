# Inkling 2 Mini on maya-ik2 — 300K fidelity arm (drift study)

Date: 2026-11-25 (bench 2026-11-24/25, +07)
Strain: `inkling-2-mini-ud-q4kxl/maya-ik2-43884ff/masterpig2-rtx4090`
Instrument: drift-study-v2-300k, frozen battery `5dff9ead0573` (168 items), thinking
pinned off, T=0, one sample per cell, static filler, single-tenant headless 4090.

Engine: [alesha-pro/maya-ik2](https://github.com/alesha-pro/maya-inkling-mini) @43884ff
(sm_89, CUDA 12.4), Unsloth UD-Q4_K_XL, FP8 KV, 344,064 ctx. Companion data: the
agent grid report (2026-11-24) and the [DSV41-1017 two-arm fidelity suite]
(../2026-11-21-dsv41-flash-two-arm-fidelity-suite.md) whose battery and protocols this
arm reuses.

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
The pinned run zeroes arithmetic (0/54) and rule_bound (0/18) at fresh, same
pinned-arithmetic phenomenon the DeepSeek arms showed.

## Thinking-on battery (separate pin, never blended)

90/168 fresh. Class profile shifts, not uniformly up: rule_bound 0→18 (fully recovered
by reasoning), conditional 14→19, tool_call 24→25; extraction 40→28 with 12 no-answer
items (reasoning consumes the 640-token budget before the VAL…END answer); arithmetic
remains 0/54.

## Tool chain (within-arm compare)

- fresh: **43/43**, both scripted failures (positions 12, 28) recovered on retry,
  zero branch failures
- at depth: **43/43** at 261,681 measured prompt tokens, 105.9 s wall, zero breaks

Reference shape (same instrument, other arm): DSV41-1017 local 43/43 at 919,234.

## Read it right

- One sample per cell; T=0 is not determinism.
- Battery calibrated on the DeepSeek family, ceiling-waived; near-miss evolution is
  the sensitivity signal, absolute scores are not cross-family comparable.
- Retention is the honest compare: Inkling 74/78 = 94.9% to 294,494 tokens; the
  DSV41 cloud arm 92/98 = 93.9% to 919,677. Different ladders, never one column.
- Isolates: `runs/2026-11-24_18000_drift-study-v2-alpha-300k.json` (+ .session.jsonl),
  `runs/2026-11-25_18000_tool-chain-40-v1.json`,
  `runs/2026-11-25_18000_tool-chain-at-depth-256064.json`.
- Ops notes: one ladder kill at the 260K rung (transient tailnet drop, 4 fast retries
  exhausted); `--resume` restored from the 200K checkpoint at zero data loss. Occupancy
  recorded per stage, all single-tenant.
