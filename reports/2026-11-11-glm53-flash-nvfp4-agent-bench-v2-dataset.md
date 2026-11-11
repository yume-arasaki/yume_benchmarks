# Agent Bench Grid v2 — the dataset: drift, cliff, contention, chain on GLM-5.3-Flash NVFP4

2026-11-11 · [@yume_arasaki](https://x.com/yume_arasaki)

The speed table for this recipe is already published ([28 Oct report](reports/2026-10-28-glm53-flash-fp4-local-two-sparks.md)). This one answers the four questions that table never does: does quality survive a long session, where does arithmetic break, do parallel agents corrupt each other's tool calls, and where does a long tool workflow fall over. Four instruments, one lane, one evening, every number below generated from frozen isolate files by a render script — the prose and the data cannot disagree because a single facts module sits behind both.

---

## WHAT WE RAN

| label | value |
|---|---|
| model | `zai-org/GLM-5.3-Flash`, NVIDIA QAD NVFP4 checkpoint |
| engine | vLLM 0.32 + MBX, TP=2 over RoCE, MTP speculative decoding |
| hardware | two DGX Spark (GB10), one active RDMA rail |
| port | 8008 |
| seats | 16 concurrent, 12 GB KV per box, 400,000-token window |
| thinking pin | `reasoning_effort: low` (no `enable_thinking` flag on this checkpoint) |
| temperature | 0 |
| battery | 168 items, sha `5dff9ead0573`, frozen |
| occupancy | sampled continuously inside every stage, depth and rung, plus a gated idle check before each latency rung |
| occupancy semantics | `running` includes the instrument's own request, so a quality stage peaking at 1 is single-tenant |

## WHAT THIS ANSWERS

- **Drift:** does quality survive a long session? → slope -80.635 items/Mtok (alpha), -87.886 (beta), neither CI includes a flat line. Loss by depth: 7 of 24 items in alpha arrive by the first rung, 13 of 27 in beta.
- **Cliff map:** at what working depth does precision bend? → it does not bend to 24 operations (158 of 160 items correct).
- **Contention:** do interleaved agents corrupt each other's tool calls? → 0 cross-talk incidents in 56 calls at up to 16 seats.
- **Tool chain:** where do long tool workflows break, and how? → 43 of 43 positions correct, taxonomy empty.

## 1. Drift across a session

6 stages per arm, depth read from the engine's own `usage.prompt_tokens` rather than a
nominal label. The battery, its sha and the canary supply are identical at every stage.

| stage | alpha depth | alpha score | beta depth | beta score |
|---|---:|---:|---:|---:|
| fresh | 92 | 165/168 | 92 | 166/168 |
| 50k | 50373 | 158/168 | 50438 | 153/168 |
| 100k | 100782 | 158/168 | 100334 | 150/168 |
| 150k | 150078 | 155/168 | 150478 | 160/168 |
| 200k | 200048 | 151/168 | 200794 | 142/168 |
| 260k | 260622 | 141/168 | 260394 | 139/168 |

- alpha slope **-80.635** items/Mtok, last-minus-first `-0.1429`, CI `[-0.1964, -0.0952]`
- beta slope **-87.886** items/Mtok, last-minus-first `-0.1607`, CI `[-0.2202, -0.1071]`
- canaries: alpha `60/108`, beta `52/107`
- item disagreement per stage: fresh 5/168, 50k 28/168, 100k 34/168, 150k 30/168, 200k 34/168, 260k 37/168

Alpha gives up 24 items: 7 arrive by the first rung (29% of the loss) and 17 across the remaining rungs. Beta gives up 27, split 13 (48%) and 14. The split is back-loaded for alpha and roughly even for beta. A single straight line through the stage points hides that split, which is why
the per-stage table sits above the slope.

## 2. Cliff map

Fresh context only, new session per item, fully parenthesised chains so the written expression
and the recorded answer agree.

| ops | correct | pct |
|---:|---:|---:|
| 3 | 10/10 | 100.0 |
| 4 | 10/10 | 100.0 |
| 5 | 10/10 | 100.0 |
| 6 | 10/10 | 100.0 |
| 7 | 10/10 | 100.0 |
| 8 | 10/10 | 100.0 |
| 9 | 10/10 | 100.0 |
| 10 | 10/10 | 100.0 |
| 11 | 10/10 | 100.0 |
| 12 | 10/10 | 100.0 |
| 13 | 10/10 | 100.0 |
| 14 | 10/10 | 100.0 |
| 15 | 10/10 | 100.0 |
| 16 | 10/10 | 100.0 |
| 20 | 10/10 | 100.0 |
| 24 | 8/10 | 80.0 |

`depth_at_50` is null — no depth in 3..24 scored at or below 50%: this checkpoint does not bend on key-verified sequential arithmetic inside 24 operations. Misses: ops=24 (8/10). Family `glm5`, tokenizer `glm5`.

This contradicts the working-precision table in the v1 drift methodology, which put a cliff
at a shallower depth. That table predates key verification against the written expression, so
the likeliest explanation is a mis-keyed ladder rather than a real cliff. Treat the v1 cliff
as retracted until re-derived.

## 3. Tool-call contention

Interleaved agents, a fixed choreography each, provably disjoint target namespaces. Each rung
ran behind a gated idle check 30.0 s apart, and every rung's gate came back idle.

| N | wall | calls correct | cross-talk | dropped | schema violations |
|---:|---:|---:|---:|---:|---:|
| 4 | 5.2 s | 8/8 | 0 | 0 | 0 |
| 8 | 6.0 s | 16/16 | 0 | 0 | 0 |
| 16 | 9.2 s | 32/32 | 0 | 0 | 0 |

Per-agent TTFT p50 spans 1454.9–1859.2 ms at N=4 and 2232.7–2901.5 ms
at N=16. The worst p99 at N=16 is 5284.8 ms, against 2346.8 ms at N=4, a factor
of 2.3. Correctness does not move across rungs.

## 4. Tool chain

43 calls, a keychain of stored values demanded again at later positions, branch points at
[8, 16, 24, 32], scripted failures after positions [12, 28].

| bucket | correct |
|---|---|
| 1-10 | 10/10 |
| 11-20 | 10/10 |
| 21-30 | 10/10 |
| 31-40 | 10/10 |
| 41-43 | 3/3 |

Break taxonomy: empty. Recovery: {'12': 'correct_retry', '28': 'correct_retry'}. Branch failures: none.

## READ IT RIGHT

- **A single sample per cell.** Every score is a single measurement at temperature 0. This
  engine is not deterministic at that setting: the arms ran the same battery at fresh context and
  returned `165/168` and `166/168`, a spread of 1 items with no
  history involved. Treat small differences as noise.
- **The battery is ceiling-limited.** It scores between 165 and 166 of 168 at
  fresh context across both arms, and every calibration before them landed in the same band. Drift
  detection rides on near-miss classes and the canary line as much as the absolute score.
- **No BF16 A/B.** No BF16 tenant exists on these boxes, so nothing here isolates how much of the
  drift or the arithmetic strength is quantization-specific.
- **The cliff map is family-bound.** A bend, or its absence, does not transfer across tokenizers.
- **Occupancy is recorded, not assumed.** Each stage samples the running-request count and KV
  fraction continuously, so co-tenant load is visible rather than claimed. The count includes the
  instrument's own request: a quality stage peaking at 1 is single-tenant, and the
  contention rungs peak at exactly their N because those are our own agents.
- **What is not measured:** memory correctness, refusal behaviour, vision, and any speed number.
  This report contains no throughput claims.

## Verdicts

- **Drift:** real in both arms — 24 items in alpha and 27 in beta across the ladder,
  split 7/17 and 13/14 between the first rung and the rest. The split is
  back-loaded for alpha and roughly even for beta.
- **Working precision:** no cliff to 24 operations, so the previously published early
  cliff does not reproduce under a key-verified ladder.
- **Contention:** 56 of 56 calls correct at up to 16 interleaved agents with
  0 cross-talk incidents, at 2.3× the latency of the smallest rung.
- **Tool chain:** 43 of 43 sequential positions correct, taxonomy empty,
  with 2 injected failures retried correctly.
