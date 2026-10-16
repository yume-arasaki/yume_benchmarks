# GLM-5.3-Flash abliterated on two DGX Sparks — pushing the 1M window (2026-10-16)

2026-10-16 · [@yume_arasaki](https://x.com/yume_arasaki)

Same lane as my [1 Oct write-up](reports/2026-10-01-glm53-flash-tensorfold-two-sparks.md): the
abliterated GLM-5.3-Flash EXL3 build on [jayleaton's TensorFold recipe](https://github.com/jayleaton/glm53-tensorfold-spark)
`@50641cf`, two Sparks, port `:8008`. That report left one question open — the window. The recipe
ships `CONTEXT=1048576`, so the "as booted" serve is already a 1M serve, and the natural next move
was to raise it and re-grid.

The raise does not exist. Not because the box runs out, but because the checkpoint says so:

```
tensorfold: --context 2097152 exceeds this model's 1048576-token window
```

That is argument validation, thirty seconds, no allocation. So this became a different report:
two full grids at the maximum window, and then a push to the actual top of it — a 1,038,602-token
prompt answered correctly. Everything below is that.

**Strain:** `glm53-flash-uncensored-exl3/tensorfold-tp2/hamster/50641cf`
**Recipe:** github.com/jayleaton/glm53-tensorfold-spark `@50641cf`, image `glm53-tensorfold:dev` (b10)
**Weights:** `neko-legends/GLM-5.3-Flash-Uncensored-EXL3` `@07135ec`, 163.6 GiB, verified on both nodes
**Serve:** `:8008`, TP=2 over the CX7 link, 4 request slots, latent FP8 KV, one shared 1,048,576-token pool, image input on (8 images a prompt, measured)
**Protocols:** night1 empty-KV lanes (T=0, thinking off), night23 needle / ctx-decode / job-at-depth / 35k towers, concurrency ladder 1→128, prose-600 with `ignore_eos`
**Configurations:** **A** as booted (`CONTEXT=1048576`), **B** a fresh boot at the same window with its own run tag, so nothing overwrites and the two are comparable
**Artifacts:** 13 isolates each, `benchmarks/runs/2026-10-16_8008_[ctxb_]*.json`

## Headline numbers

| | |
|---|---|
| Empty context, C1 peak (A) | structured count-200 **103.9** · prose-600 **54.7** · code clamp **93.3** · json catalog **79.7** |
| Empty context, C4 peak (A) | structured **174.4** · prose **86.2** · code **143.4** · json **158.5** |
| Needle, 3 placements × 6 depths | **36/36** across two boots, deepest at 874,673 tokens |
| Decode at the top of the ladder | 37.2 (A) / 37.7 (B) tok/s at 874k, from 49.0 / 47.9 at 8k |
| Prefill across the ladder | 1,554 → 1,130 tok/s, TTFT 5.2 s → 773 s |
| **The actual window** | **1,038,602-token prompt accepted, all three placements returned, 944 s** |
| Concurrency, 4 slots | aggregate peaks at C4 (84.1) then plateaus ~42–47 to C128 |
| Tools | real `tool_calls`, no XML in content, both configs |
| Garble | 1 cell in A (the OMP 35k tower), **0 in B** |

## Empty context, twice

Two runs a lane, per-protocol. A and B are the same configuration, so the spread between them is
this harness's noise floor on this lane — worth printing because it is what makes a single-run
"improvement" unreadable.

| Lane | A C1 | B C1 | A C2 | B C2 | A C4 | B C4 |
|---|---:|---:|---:|---:|---:|---:|
| Count from 1 to 200 | 103.9 | 105.6 | 152.5 | 151.2 | 174.4 | 179.2 |
| Explain a hash map (600 tok) | 54.7 | 58.0 | 69.7 | 71.1 | 86.2 | 88.3 |
| Fifty identical Python clamps | 93.3 | 94.6 | 127.4 | 141.7 | 143.4 | 193.8 |
| A JSON blob of fake GPU stats | 79.7 | 74.9 | 124.4 | 122.4 | 158.5 | 158.4 |

Against the 1 Oct grid on this same strain: 109.1 / 57.7 / 96.5 / 80.2 at C1. Every lane lands
within a few percent either side, with clamps and JSON showing the widest run-to-run spread at C4.
Two weeks on, same recipe sha, same ballpark.

`2^10 + 3^5` thinking off: exact_match **0**, both configs. Thinking on: **1**, both configs. That
cell has now fooled GLM, DSV41, two 27B quants, MiMo and hibrid48, and I have written before that
it is the probe. Both clocks ran here and both agreed across two boots, so the ledger stays
probe-suspect, not model-suspect.

Tools: `get_weather`, `tool_calls` present, `{"city":"Tokyo"}`, `finish_reason=tool_calls`,
12 completion tokens, no XML. Identical in A and B.

## Long context: the ladder

Needle planting is the part people misread, so to be explicit: three placements per depth — 5%, 50%
and 95% of the context, each a distinct six-hex code — and `3/3` means all three positions were
retrieved. Two boots:

| I aimed at | A actual | A hits | B actual | B hits | A prefill | B prefill | A TTFT | B TTFT |
|---:|---:|---|---:|---|---:|---:|---:|---:|
| 8k | 8,085 | 3/3 | 8,056 | 3/3 | 1,554 | 1,528 | 5.2 s | 5.3 s |
| 32k | 32,548 | 3/3 | 32,527 | 3/3 | 1,617 | 1,624 | 20.1 s | 20.0 s |
| 128k | 129,123 | 3/3 | 129,125 | 3/3 | 1,554 | 1,557 | 83.1 s | 82.9 s |
| 256k | 257,651 | 3/3 | 257,698 | 3/3 | 1,461 | 1,468 | 176.4 s | 175.6 s |
| 512k | 514,001 | 3/3 | 513,784 | 3/3 | 1,303 | 1,311 | 394.6 s | 391.8 s |
| 900k | 873,955 | 3/3 | 874,673 | 3/3 | 1,130 | 1,143 | 773.1 s | 765.5 s |

36 of 36. The 1 Oct grid got 18 of 18; this is that result reproduced on two separate boots.

Decode after a 256-token forced fill, same depths:

| Depth | 8k | 32k | 128k | 256k | 512k | 874k |
|---|---:|---:|---:|---:|---:|---:|
| A | 49.0 | 51.0 | 40.4 | 51.1 | 41.5 | **37.2** |
| B | 47.9 | 44.8 | 42.8 | 44.8 | 40.6 | **37.7** |

24% down at the top from the 8k lane in A, 21% in B, with wobble in between as large as the trend
itself — the sag at 128k and 512k in A appears at different depths in B, so I read it as noise on a
256-token sample rather than a depth effect. The 1 Oct ladder read 46.4 → 40.1 over the same span,
so the top rung is a few points lower this time. Printed, not explained.

The generation itself stays coherent at every depth. At 874k both boots answer a RoCEv2 question
with a full 256-token numbered list, correct content, `finish_reason=length`, no repetition
collapse. Retrieval working is one thing; the model still writing like itself a million tokens in
is the more useful signal.

## The actual window

The grid's top rung lands at 874k actual, which is 83% of the window. A reported limit is weak
evidence, so I drove the top of it directly. The probe measures the filler's cost with the engine's
own tokenizer (49.4375 tokens a unit), builds a prompt a thousand tokens short of the window, and
plants three distinct phrases at 5/50/95% of it:

```
probe: 3164 prompt tokens for 64 filler units -> 49.4375/unit
built 21194 filler units (~1047778 tokens est); target 1048000
accepted: prompt_tokens=1038602 completion_tokens=112 elapsed=944.0s
prefill_rate_report=1100 tok/s (prompt+decode, wall clock)
reply='KIWI-9472-EMBER\nPLUM-3310-TIDE\nONYX-8065-VALE'
MAXCTX_PROMPT_OK prompt_tokens=1038602 needles=3/3
```

1,038,602 tokens in one request, three placements across the whole span, returned in order, 112
tokens out, 15 minutes 44 seconds wall clock. A separate single-needle run at the same target
returned 1,038,558 tokens with the middle phrase. Same shape the ladder tests at 874k, confirmed at
the window.

**The number to watch is memory, not quality.** I sampled `MemAvailable` on both nodes across that
prefill, 73 samples over 16 minutes. Head slid from 9,975 MiB to a floor of **8,562 MiB**; worker to
11,058 MiB. The kit's own documented floor is 8 GiB, so the head node finishes a full-window prefill
with roughly **0.4 GiB of that buffer left**. It held with zero errors, but a full-window request
plus anything else heavy on the head is where this breaks. The shared 1,048,576-token KV pool is
what stops a second full-window request being admitted at all.

## Concurrency, full ladder

Prose-600 with `ignore_eos`, aggregate decode, 4 slots (`GLM53_TF_BATCH=4`). Three passes — A, and B
swept twice:

| Streams | A (ok) | B pass 1 (ok) | B pass 2 (ok) |
|---:|---:|---:|---:|
| 1 | 47.4 (1/1) | 49.3 (1/1) | 48.6 (1/1) |
| 2 | 64.2 (2/2) | 63.9 (2/2) | 61.7 (2/2) |
| 4 | **84.1 (4/4)** | **84.1 (4/4)** | **81.3 (4/4)** |
| 8 | 53.3 (8/8) | 54.6 (8/8) | 51.2 (7/8) |
| 16 | 42.7 (13/16) | 47.2 (16/16) | 46.7 (16/16) |
| 32 | 43.0 (26/32) | 44.5 (24/32) | 45.8 (20/32) |
| 64 | 42.3 (29/64) | 42.9 (30/64) | 44.0 (23/64) |
| 128 | 42.5 (60/128) | 42.6 (51/128) | 42.0 (66/128) |

Four seats, hard ceiling at ~81–84, then a plateau at ~42–47 while TTFT grows into the minutes
(205 s median at C128 in A). The streams that do not finish end with `exc_ConnectionResetError` —
that is the engine shedding queued requests under pressure. `/health` read `errors: 0` throughout
and no stream ever produced garble. The 1 Oct report used a counting sweep with a short ladder; this
is the full ladder on prose, and the shape is the same.

## The 35k towers, and an intermittent catch

About 35k in the prompt, tools attached, one turn each. Not the real GUIs.

| Tower | Config | Prompt tokens | Completion | Result | tok/s |
|---|---|---:|---:|---|---:|
| OMP | A | 35,072 | 2,048 | `finish_reason=length`, XML dumped in content, no `tool_calls` | 68.2 |
| OMP | B | 35,114 | 32 | `tool_calls` present, clean, no XML | 43.7 |
| Hermes | A | 36,332 | 63 | `tool_calls`, clean | 51.3 |
| Hermes | B | 36,332 | 64 | `tool_calls`, clean | 52.8 |

This is the honest catch of the night. In A, the OMP tower wrote `<tool_call>write_file<arg_key>path</arg_key>...`
into content and ran to the token cap; in B, the identical prompt returned a clean 32-token tool
call. Same strain, same boot configuration, same protocol, two boots. So that failure mode is
**intermittent**, not a fixed property of the lane, and a single grid pass over that cell cannot
tell you which behaviour you get. The 1 Oct pass on this same tower recorded the same shape as A —
a full 2,048-token write with the save attempted after — so A reproduces October and B is the odd
one out. Two boots is not a rate; what it does mean is that the clean result I published in October
is not something to build on. The B OMP row is a 32-token turn, so its 43.7 is not a speed row.

## Job at depth, and a two-point artifact

| Job | A @32k | A @128k | B @32k | B @128k |
|---|---:|---:|---:|---:|
| structured (count) | 92.5 | 97.1 | 89.5 | 88.2 |
| prose | 48.6 | 50.2 | 49.7 | 48.0 |
| code (clamp) | 84.2 | 82.4 | 75.3 | 84.1 |
| tools call | 36,423.5 | 31,358.9 | 4,536.1 | 62,162.6 |

Real `tool_calls` at both depths in both boots. The tool rows are 12-token turns and their tok/s
numbers are two-point timing artifacts — the 1 Oct report flagged the same thing, so it is in the
table rather than hidden in the JSON. Ignore that row; keep the fact that the call happened.

## Energy

GPU rail, both Sparks summed, `nvidia-smi`. Night1 structured C1: 478.3 J over 4 samples, mean pair
power 75.1 W, **1.19 J/completion-token** (B: 507.5 J, 73.6 W, 1.27 J/tok). Concurrency sweep
J/completion-token: 2.34 at one stream, 1.38 at four, ~1.45 hovering through the plateau to C128.
Four streams is both the fastest and the most efficient point on this box, which is the whole
argument for the 4-slot config.

## What it cost me

Prefill at the top end. 1,130 tok/s and 773 s to first token at 874k; 944 s for the full-window
prompt. Nothing about that is interactive. The prefill rate also drifted down across the night
versus 1 Oct (1,554 → 1,130 here against 1,497 → 1,320 there), and I have no measurement that
separates engine behaviour from thermal state on the second rack-hour.

Four seats, hard. Aggregate falls to ~42 past C4 and recovering it means changing `GLM53_TF_BATCH`,
which changes the memory budget.

The memory floor at the window. 0.4 GiB above the kit's own documented 8 GiB, measured, on the head
node only.

## What I did not prove

1. **Retrieval is not reasoning at depth.** Finding three planted strings at 5/50/95% proves the
   indexer and KV path work across the span. Comparing or aggregating facts a million tokens apart
   is the harder task and is untested here.
2. **The filler is one sentence repeated ~21k times**, which is the easy case for a sparse-attention
   indexer. The only heterogeneous long-context cells in this grid are the two 35k towers.
3. **n is small at the top.** One request per depth, twice. No error bar at 874k beyond those two.
4. **Concurrency was measured at short context only.** Every sweep rung carries ~600 tokens, so the
   interaction between the four slots and a full KV pool is unmeasured.
5. **Long-context tool calling is untested past 128k**, and the XML-vs-tool-call instability above
   is only characterised at 35k.

## Client side

This lane is wired into my OMP install as the default model role, and that took a fix: OMP 18.1.21
hard-crashes on catalog build since its remote catalog started listing a Claude model its embedded
rules cannot classify (`AmbiguousOverlapError` on `claude-haiku-5-5`). Two config-level changes
cleared it — excluding the provider, and dropping that one cached catalog row. The lane itself is
clean from the client: exact-echo runs, a real bash tool round, and the serve counter confirming
which endpoint answered. The compaction budget is set to 200k deliberately, because this lane's own
TTFT numbers say a turn at 257k already costs three minutes.

## Isolates

- `2026-10-16_8008_sparkdash-{structured,prose,code,json}*` — empty-KV lanes, A, 2 reps a concurrency
- `2026-10-16_8008_needle-5-50-95`, `_ctx-decode-256`, `_job-at-depth`, `_omp-workload-35k`, `_hermes-workload-35k`
- `2026-10-16_8008_sparkdash-conc-sweep-v1` — full ladder 1→128
- Same 13 files with the `-ctxb` tag for configuration B, plus
  `2026-10-16_glm53tf_ctxb_sweep-parent-strain.json` and the probe logs under `raw-data/`

## Credit

Engine: TensorFold. Recipe: [jayleaton/glm53-tensorfold-spark](https://github.com/jayleaton/glm53-tensorfold-spark)
`@50641cf`. Weights: [neko-legends/GLM-5.3-Flash-Uncensored-EXL3](https://huggingface.co/neko-legends/GLM-5.3-Flash-Uncensored-EXL3),
abliterated build on top of Orca Router's abliterated model, upstream GLM-5.3-Flash by Zhipu.
Drafter: `incoai/GLM-5.3-Flash-DFlash2`. The numbers are mine.
