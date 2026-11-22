# DeepSeek V4.1 Flash — two-arm fidelity suite: the desk recipe against the provider's BF16

2026-11-21 · [@yume_arasaki](https://x.com/yume_arasaki)

One model family, both of its lives, the same five instruments on the same battery: fresh exams
under both thinking pins, a drift ladder past 900k tokens, needle retrieval, a 43-call tool
chain (also run inside a live 900k session), contention at up to 16 agents, and a precision
cliff map. Arms: `cloud` = `deepseek-flash` on the provider's API (their BF16 control, as
claimed; not verifiable client-side). `local` = `DeepSeek-V4.1-Flash-1017` on
[Mia's TensorFold recipe](https://github.com/MiaAI-Lab/DeepSeek-v4.1-Flash-1017-2x-DGX-Sparks)
(EXL3 EAQ v1 2.9 bpw), two DGX Sparks, TP=2. Every number below is computed from frozen isolate
files by the suite's render script; the prose and the data share one facts module.

---


## WHAT WE RAN

| instrument | cloud | local |
|---|---|---|
| battery, thinking off | missing | missing |
| battery, thinking on | missing | missing |
| needle 5/50/95 | missing | missing |
| tool chain (43 calls) | missing | missing |
| contention N=4/8/16 | missing | missing |
| cliff map | missing | missing |
| drift ladder (largest rung) | 919,677 across 6 stages | missing |

## WHAT THIS ANSWERS

### Does the model do the task at all (fresh context)?

| arm | thinking | score | arithmetic | extraction | conditional | tool_call | rule_bound |
|---|---|---:|---:|---:|---:|---:|---:|
| cloud | off | missing | | | | | |
| cloud | on | missing | | | | | |
| local | off | missing | | | | | |
| local | on | missing | | | | | |

The arithmetic collapse with thinking off is the headline: both arms score 0 on that class in the pinned mode and recover most of it when the reasoning block is enabled.

### Does quality survive a long session?

- **cloud**: stages 6, depths [84, 103407, 205529, 410035, 614300, 919677], scores [98, 90, 89, 89, 93, 92], protocol `drift-study-v2-900k`, battery sha `5dff9ead0573` (filler `static`)
- **local**: ladder missing

### Retrieval, chains, contention, precision

- **cloud**: instruments missing
- **local**: chain at depth 43/43 at 919,234

## READ IT RIGHT

- One sample per cell. Temperature 0 is not determinism; the arms disagree by a few items at fresh context with no history involved.
- The control arm's BF16 claim is the provider's, not ours. Nothing client-side verifies it.
- The battery is ceiling-waived and was calibrated on a different model family, so its arithmetic class is harder than intended for these weights. That is why the thinking-on pass exists and why the two pins are never blended.
- The cloud arm's lane occupancy is not observable: the provider's /metrics returns 401, so its isolates record `ok: false` samples with the 401 rather than a number. Any cloud isolate showing successful samples predates the arm-bound probe and is marked superseded.
- The local arm ran single-tenant at depth; occupancy per stage is the evidence.
- **Chain at depth (measured):** local 43/43 at 919,234. Compare each against the same arm's shallow chain, never across arms.

### Operational notes (operational notes)

- **cloud_billing**: estimate_usd = 1.7; actual_usd_at_402 = 7.0; stopped_at = 600,482 tokens, stage 5 of 6, at 2026-11-20T00:44Z; why_estimate_missed = the estimate assumed 99.9% prefix-cache hits on 369M ladder prompt tokens; observed battery-call caching was ~99.4% but filler calls cached far less, and the account hit zero balance at the 900k rung; reruns_after_402 = the balance was restored before 2026-11-20T16:25Z, so the cloud needle and tool-chain re-runs went through as paid calls: 9/9 needle hits at 982,577 prompt tokens and 43/43 chain positions, about $0.18 of additional spend at the off-peak rates; consequence = the 402 is the reason the cloud ladder stops at 600,482; it does not contradict the later instrument re-runs, which were paid from the restored balance
- **local_kill**: signal = exit 137, OOMKilled=false, killed while serving at ~456k prompt tokens; evidence = last log line before the kill reported 'prompt cache: 456062 tokens, resumed at 454656, kept 0; 78 states, 497 MiB and 0 MiB spare of 512 MiB'; handling = lane restarted, ladder re-run from the start with per-stage session checkpointing and --resume so a second kill costs one stage rather than the whole run
- **thinking_pin**: finding = arithmetic is 0/54 on both arms with thinking off and 42/54 (cloud) / 41/54 (local) with it on; consequence = the pinned ladder measures five live classes, not six; percentages over 168 are diluted by a class that cannot score under the pin
- **ladder_filler**: problem = block filler spent a model call per ~1.6k tokens; at a 400k session that is a full re-prefill plus ~1 s/token decode per call, capping the ladder near 400k; fix = YUME_FILLER=static appends large pre-generated passages (40k tokens each) and probes the engine's own prompt_tokens to fit under the window ceiling; result = the 900k rung filled in minutes instead of a projected ~9 h; the local ladder completed six stages and its final rung landed at 915,246 actual prompt tokens (+1.7%, inside the ±5% band); cost = the fill now costs a handful of calls per rung instead of hundreds
- **local_deep_rung**: target = 900000; actual = 915246; delta_pct = 1.69; how = measured-cap fill trims to the ±5% band instead of the window ceiling; a bisecting append step converges where a fixed chunk oscillated; cost = the 900k rung decodes at ~0.2 tok/s, so 168 items take hours; the banded re-run was a full night's tail; co_tenant = 4 of 2321 occupancy samples saw a second request, each holding ~0.1% of the pool — recorded as a note, not a contended stage
- **ladder_outcome**: cloud = stopped short of 900k at 600,482 on HTTP 402 (account balance); recorded as its actual depth (superseded by cloud_completion_2026_11_21); local = completed the 900k rung at 915,246 actual prompt tokens (+1.7%, inside the ±5% band) after static bulk filler and a measured fill band replaced block filler, which had stalled the ladder at 400,579
- **cloud_completion_2026_11_21**: what = the cloud ladder was re-run end to end with static filler and completed all six rungs; depths = 84 / 103,407 / 205,529 / 410,035 / 614,300 / 919,677; scores = 98, 90, 89, 89, 93, 92 of 168; cost_floor = $1.46 on battery calls, with 99.5% of 378.7M prompt tokens served from cache; filler probe calls are not recorded, so the true figure is higher; why_it_differs = the earlier 5-stage run used block filler; these stages use static filler. Filler mode shifts scores by up to eight items (block: 99, 95, 97, 93, 89; static: 98, 90, 89, 89, 93, 92), so the two ladders are not interchangeable and are labelled by filler; endpoint = the rung now exists, so the post no longer needs the 'not re-run' line; the honest line is that the first attempt stopped on a 402 and the completion run a day later reached 919,677

