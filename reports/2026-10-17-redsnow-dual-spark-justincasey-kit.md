# RED-SNOW-5.3-FLASH abliterated on two DGX Sparks — justin's TensorFold kit, verified (2026-10-17)

2026-10-17 · [@yume_arasaki](https://x.com/yume_arasaki)

Third write-up for this checkpoint, first one where it runs on **two** Sparks. The
[single-Spark pairs](reports/2026-10-12-redsnow-one-spark-receipts.md) left the dual-node question
open, and my own ledger said the answer was a different kit: RED-SNOW's SAGE MixedK layout cannot
load on a vLLM EXL3 path, and [justincasey's
recipe](https://github.com/justincasey/RED-SNOW-5.3-Flash-2x-DGX-Spark) loads it on TensorFold plus
a `redsnow-v1` branch that adds the half-step bit widths, the mul1 codebook and the trellis/tail
fixes. I had read that kit on 2026-10-16 and recorded it as **evaluated, not loaded**. This is it
loaded, gated, gridded, and pushed to a million tokens.

The short version: **his numbers are real.** Drafted decode mean 39.45 against his published 39.57,
undrafted 24.52 against 24.44, long-context within a few percent in both directions, and every one
of his own gate checks passing 22/22 and 31/31. He is also right about the thing that makes this
engine unusual, and I have better evidence for it than he did.

**Strain:** `redsnow-sage-exl3-4.91bpw/tensorfold-tp2/justincasey/f5d727a`
**Kit:** github.com/justincasey/RED-SNOW-5.3-Flash-2x-DGX-Spark `@a0817ab`, engine TensorFold `redsnow-v1.0.2` (`f5d727a`)
**Weights:** `vcruz305/RED-SNOW-5.3-FLASH-EXL3-SAGE-4.91bpw` @ `f6b89637`, 36 shards, 186 GiB; `lm_head` from `Blackfrost-AI/RED-SNOW-5.3-FLASH-BF16` (sha256-pinned, verified)
**Drafter:** `incoai/GLM-5.3-Flash-DFlash2` @ `bf582e4e` (CC BY-NC-ND 4.0 — non-commercial research on my own desk)
**Serve:** spark-1 `:8012`, TP=2 over the QSFP link, served id `RED-SNOW-5.3-Flash`, kit-default `CONTEXT=131072`, fp8 latent KV — then 1,048,576 on the q4 latent
**Protocols:** his `bench all` (thinking on, greedy) and the house agent grid (thinking off, T=0), kept apart

## Headline

| | |
|---|---|
| His 6-prompt drafted decode mean | 39.57 published → **39.45 measured** |
| His 6-prompt undrafted mean | 24.44 published → **24.52 measured** |
| His gate suites | **smoke 22/22, decode 31/31**, every drafted reply equal to the pinned serial ids |
| Long context, 127,712-token prompt | 330.9 s TTFT, 386 tok/s prefill (his: 336 s, 381) |
| Concurrency 1 → 128 streams | wall aggregate **22.6 → 19.5 tok/s** — flat, and every stream still completed |
| Retrieval | 3 placements × 3 depths = **9/9 exact** at the default window; 3/3 at 1,039,849 tokens |
| The window | `CONTEXT=131072` default; **1,048,576 on the q4 latent** (fp8 is refused) |

## What the kit is

One script with `preflight/install/download/view/start/stop/status/smoke/bench`. `install` clones
his TensorFold branch and compiles five CUDA extensions on **each** node; `download` fetches the
pack, the BF16 `lm_head` shard and the drafter; `view` links the pack and swaps the head; `start`
launches both ranks over the QSFP link and waits for the API. Rank 0 serves the OpenAI API and
handles the rendezvous.

It is a product in a way the earlier dual-node attempt here was not. My 2026-10-09 run of this
checkpoint on `vcruz305`'s multinode pipeline was 8.63 tok/s over TCP with NCCL blocked; this kit
reaches **39.45 tok/s drafted** on the same pair with no transport work from me at all. What it
does not bring is batching: see the queue section.

## His numbers, mine beside them

`./redsnow.sh bench all`, 30 m 43 s, both gate suites passing. His figures are from his README and
his own `bench/records/2026-10-10-v1.0.2-baseline/`:

| Measurement | His published | Ours | Delta |
|---|---:|---:|---:|
| Decode, DFlash2 drafted mean over his 6 prompts (512 tok each, thinking on) | 39.57 tok/s | **39.45 tok/s** | −0.3% |
| Decode, same prompts without drafting | 24.44 tok/s | **24.52 tok/s** | +0.3% |
| TTFT, ~220-token prompt | 0.78 s | **0.99 s** | +27% |
| TTFT, 127,712-token prompt | 336 s | **330.9 s** serial, 332.5 s drafted | −1.5% |
| Prefill at 127.7K | 381 tok/s | **386.0 / 384.1 tok/s** | +1.3% |
| Decode after that prompt, drafted / not | 34.9 / 22.7 tok/s | **33.42 / 23.18 tok/s** | −4.2% / +2.1% |
| SixCat decode p50 (his synthetic ceiling) | 67.37 tok/s | **66.13** (p95 67.10, max 67.17) | −1.9% |
| SixCat prefill p50 | 378.6 tok/s | **388.23** (p95 389.07) | +2.5% |
| SixCat balanced aggregate @ C1 | 36.17 tok/s | **36.16 tok/s** | −0.03% |

Per prompt, drafted (median of 3 reps): code 39.80 (his 39.71), prose 32.03 (31.93), math 55.71
(55.85), chat_explain 34.44 (34.81), chat_multiturn 35.04 (35.14), long_doc 39.71 (39.95).

The one figure that is clearly off is the short-prompt TTFT, 0.99 s against his 0.78 s, and it is the
noisiest item on the list: three prompts, one sample each, and the prose prompt's 1.17 s carries its
own prefill variance. Everything else is inside 4%, most of it inside 1%. The undrafted mean
reproducing to 0.3% is the one I trust most, because it needs no drafter acceptance luck.

**His gates, verbatim from the run:** `ALL CHECKS PASSED: 22/22` on smoke — `/tokenize` prompt ids
equal to his pinned ids on six prompts, thinking-off ends `<|assistant|>` with no markup leak,
thinking-on streams round by round (7 deltas for 21 tokens, largest gap 0.089 s), **a drafted reply
equals the `"draft": false` one token for token**, DFlash2 is the default drafter, and an invalid
`tf_policy` is refused. Then `ALL CHECKS PASSED: 31/31` on decode, where every drafted reply equals
the serial ids pinned in `bench/redsnow_reference.json` (`first diff None` everywhere).

### The short-prompt anomaly

Worth one line because it is the kind of thing that gets quoted as a win: his `prose` prompt is the
slowest of the six at 32.03 tok/s against 55.71 for `math`. That is not variance. DFlash2's
acceptance rate tracks how predictable the continuation is — math drafts 4.71 tokens a round, prose
2.19 — and he says the same thing in his README (31.9 prose against 55.9 math). Drafting is a bet on
the text, and prose pays the vig.

## The default-window grid

The house agent grid on the kit's own default window (`CONTEXT=131072`, thinking off, T=0), 13
isolates, all of them carrying this strain.

| Lane | C1 | C2 | C4 |
|---|---:|---:|---:|
| structured count-200 | 62.6 | 67.1 | 68.6 |
| prose-600 | 36.9 | 39.7 | 38.3 |
| code clamp | 61.6 | 62.6 | 63.7 |
| json catalog | 26.2 | 27.3 | 24.2 |

Those are **wall-clock aggregates**, not the per-stream sums the harness also writes — on this
engine the two disagree in a way that matters, which is its own section below. Garble: **0** in every
cell at this window — the raised-window run has one exception, and it is the OMP tower. Tools: real
`tool_calls` with `{"city":"Tokyo"}`, no XML in content.

At depth: retrieval was exact on all three placements at 8k, 32k and 131,072, prefill 352–398 tok/s,
TTFT 23.5 s → 329.0 s. The frozen ladder reaches 900k, so the rungs above the window came back as
refusals, recorded as rows with the server's own message — *"This server's maximum context length is
131072 tokens"* — rather than crashing the phase. The two 35k towers came back clean, including the
OMP tower that produces XML-in-content intermittently on the GLM lane sharing this pair.

## He is right about the queue

His README says the engine serves one request at a time: *"An earlier SixCat run with 1, 2, 4 and 8
requests in flight measured the same ~35.9 tok/s aggregate at each level: requests queue, so `bench
sixcat` stops the concurrency curve at one."* He tested four levels. The full ladder runs eight:

| Streams | wall aggregate | per-stream decode | streams ok | TTFT median |
|---:|---:|---:|---:|---:|
| 1 | 22.64 | 36.25 | 1/1 | 5.0 s |
| 2 | 24.77 | 39.12 | 2/2 | 8.8 s |
| 4 | 21.88 | 39.71 | 4/4 | 25.1 s |
| 8 | 21.59 | 41.51 | 8/8 | 52.9 s |
| 16 | 20.25 | 39.91 | 16/16 | 114.0 s |
| 32 | 19.84 | 39.70 | 32/32 | 238.1 s |
| 64 | 19.77 | 39.89 | 64/64 | 485.5 s |
| 128 | 19.51 | 39.51 | **128/128** | 978.5 s |

**Confirmed, and then some.** Aggregate throughput does not grow across a 128× increase in offered
concurrency — it sags slightly — while per-stream decode never leaves 36–41 and first-token latency
climbs to sixteen minutes. Every stream still completed, with no failures and no reset requests, so
what concurrency buys here is queue position, not throughput: a hundred and twenty-eight clients
share the ~20 tok/s that one client gets.

Energy per completion token *falls* from 3.30 at one stream to ~2.67 from C4 up, then holds. That is
the idle time between rounds being filled in, and it is the real argument for sending concurrent
work at this engine: not more tokens a second, but less electricity a token.

**A reading trap worth publishing**, because I nearly fell into it. The house grid also writes
`decode_agg`, the *sum* of per-stream rates. On the night1 cells that column reads 68.6 / 142.8 /
286.1 for structured at C1/C2/C4 — the shape of perfect scaling — while the same cells move 62.6 /
67.1 / 68.6 tokens a second of wall clock. Nothing is scaling; the column is adding up four
serialised streams. I have left both columns in the findings doc so nobody quotes the wrong one.

## Pushing to the window

The kit's `CONTEXT=131072` is **prompt + reply**, so a 262k-token rung is a 400, not a slow answer.
His capacity table says the fp8 latent stops well short of 1M and the q4 latent is what fits
(110.50 GiB a rank of ~113.5). Both halves of that are true here:

```
# fp8 at 1,048,576 — refused
tensorfold: CUDA startup memory budget cannot fit requested context 1048576; estimated largest
fitting prompt-plus-reply window: 1031720 tokens across the ranks. … KV precision is unchanged.

# q4 at 1,048,576 — serves
06:58:30 serving RED-SNOW-5.3-Flash at http://10.0.0.1:8012/v1 (ready in 147 s)
```

The server's own `context_length` went from 131072 to **1048576**, and the exact-echo smoke still
passes at the raised window. The engine accepts `--kv-dtype bf16, int8, int4, fp8, q4`; the kit passes
`KV_DTYPE` straight through, though its `.env.example` only documents the first two.

Prompt at the raised window:

```
    calibration: 3333 tokens for 64 units, 50373 for 1024 -> 49.0000/unit, overhead 197 tokens
    built 21216 filler units (~1039781 tokens est); target 1040000
    accepted: prompt_tokens=1039849 completion_tokens=108 elapsed=2898.3s
    prefill_rate_report=359 tok/s (prompt+decode, wall clock)
    reply='KIWI-9472-EMBER\nPLUM-3310-TIDE\nONYX-8065-VALE'
    MAXCTX_PROMPT_OK prompt_tokens=1039849 needles=3/3
```

A **1,039,849-token prompt**, three phrases planted at 5/50/95% of it, all three returned in order,
48 minutes and 18 seconds of wall clock. That is 99.2% of the window. It is the second million-token
prompt I have had answered on this pair — the [GLM lane](reports/2026-10-16-glm53-flash-abliterated-1m-two-sparks.md)
did 1,038,602 on 2026-10-16 — and the first for this checkpoint. Worth one aside on method: my first
attempt at this prompt came out 986,145 tokens because a single small calibration probe inflates the
per-unit cost (it measured 52.08 tokens a filler unit; the true slope is 49.00, the 197-token fixed
overhead being what inflates a 64-unit probe and then dilutes to nothing across 20,000 units). Two
calibration points fix it, and the second run landed within 151 tokens of its target.

Memory is the honest caveat. While that prompt prefilled, `MemAvailable` bottomed at **662 MiB on the
head node and 936 MiB on the worker** across 120 samples — the q4 latent at a full window leaves
almost nothing. It held, but this is not a configuration with room for a second large request, and it
is the number I would watch before calling a million tokens "supported" on this pair.

## The same grid at a million

With the window at 1,048,576 the frozen ladder stops refusing and starts reporting:

| Cell | 8k | 32k | 131k | 262k | 512k | 874k |
|---|---:|---:|---:|---:|---:|---:|
| needle, 3 placements | 3/3 | 3/3 | 3/3 | 3/3 | 3/3 | 3/3 |
| TTFT | 23 s | 85 s | 328 s | 671 s | 1,343 s | 2,392 s |
| ctx-decode tok/s | 29.7 | 30.8 | 30.5 | 28.3 | 28.8 | 28.0 |

At the default window those last three columns were 400s. Here they are exact recall at 874,495
tokens, and decode holds 28.0–30.8 across a 107× span of context. The empty-context lanes are
unchanged by the switch (structured C1 62.5 wall against 62.6 on fp8, prose 36.6 against 36.9), so the
q4 latent buys the window without a short-context tax — which is the opposite of what I expected from
a narrower cache.

Work at depth, and the towers:

| Cell | 32k | 131k |
|---|---:|---:|
| job: structured count | 59.9 | 62.5 |
| job: prose | 30.0 | 38.5 |
| job: code clamp | 52.3 | 45.7 |
| job: tools call | real `tool_calls` | real `tool_calls` |

Prefill held 373–395 tok/s throughout, and every one of those cells is garble-free.

**The OMP tower is the one failure of the night, and it is the interesting one.** At the raised
window `omp@35k` produced 2,048 tokens of XML in content with `finish_reason=length` and no
`tool_calls` — garbled, `xml_tool`. The identical prompt at the default window had come back clean.
`hermes@35k` was clean at both. So the tool-call instability I reported on the GLM lane earlier this
week is **not a GLM property**: same prompt, same engine family, same pair, one clean pass and one
garbled pass. Two passes is not a rate, but it does mean the clean result I published for this cell
at 131k should not be quoted as though it were stable.

Concurrency at the raised window:

| Streams | 1 | 2 | 4 | 8 | 16 | 32 | 64 | 128 |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| wall aggregate | 28.27 | 22.30 | 21.21 | 20.38 | 19.99 | 20.06 | 19.64 | **19.75** |
| per-stream decode | 36.0 | 36.3 | 38.1 | 39.3 | 39.6 | 40.2 | 39.7 | 40.0 |
| streams ok | 1/1 | 2/2 | 4/4 | 8/8 | 16/16 | 32/32 | 64/64 | **128/128** |
| TTFT median | 2.3 s | 10.3 s | 25.0 s | 56.0 s | 118.0 s | 237.4 s | 483.9 s | 964.3 s |

Same shape as the fp8 default window: the aggregate does not move, the queue grows, and every stream
is still answered. The window change costs nothing here either.


## What it cost me

The kit needs **~200 GiB free a node** and my head node had 5.1 GiB. Reclaiming regenerable caches
only — one untagged 34.4 GB image, the build cache, `~/.cache/vllm`, `~/.cache/pip` — got it to 41
GiB, and the pack was reused through a symlink farm instead of re-downloaded, so the real cost was
about 8 GiB of venv and compiled kernels. That worked. It is worth saying that it worked because the
staged pack's revision and directory name happened to match the kit's pins exactly; a checklist that
assumed "just download it again" would have failed on this desk.

Two desk-level things the kit assumes and this pair does not have: the **same username and home
layout on both nodes** (fixed with one symlink), and **egress on both nodes** (rank 1 had no default
route at all — its WiFi profile was down — so the first `install` died on `Could not resolve host:
github.com`). Neither is the kit's fault; both cost time.

## What I did not prove

1. **The queue is measured, not explained.** I can show throughput is flat to 128 streams and that no
   stream failed. I have not read the engine's scheduler, so I cannot say whether that is one decode
   slot by design or a configuration choice.
2. **Thinking-off grid cells are not comparable to his thinking-on bench numbers.** The house
   protocol pins thinking off (this serve is thinking-on by default: 21 completion tokens against 3
   for the same answer). I kept the two protocols apart deliberately and labelled which is which.
3. **One node pair, one cable, one night.** Nothing here says how the numbers move with a different
   interconnect or a warmer box.
4. **The 1M configuration is a different KV dtype from his measured path.** It serves and it
   retrieves, but its decode and prefill numbers at long context are not the ones he published, and
   the grid at that window is reported separately.

## Isolates

- `2026-10-17_8012_*` — 13 isolates, kit-default window: `sparkdash-{structured,prose,code,json}`,
  `needle-5-50-95`, `ctx-decode-256`, `job-at-depth`, `omp-workload-35k`, `hermes-workload-35k`,
  `tools-omp-v1`, the two maths clocks, and `sparkdash-conc-sweep-v1`
- `2026-10-17_8012-ctx1m_*` — the same three phases at 1,048,576
- His own records: `~/src/redsnow-justin/bench/runs/2026-10-17-041107-all/` and
  `bench/records/2026-10-10-v1.0.2-baseline/`

## Credit

Kit and engine branch: [justincasey](https://github.com/justincasey) (`redsnow-v1` over TensorFold
0.6.5, with `vcruz305`'s `glm53-tp4-spark` CUDA work and MiaAI-Lab's ports, all credited in his
README). Weights: `vcruz305`'s SAGE MixedK 4.91bpw quant of Blackfrost-AI's abliterated
RED-SNOW-5.3-FLASH, upstream GLM-5.3-Flash by Zhipu. Drafter: `incoai`'s DFlash2, CC BY-NC-ND 4.0 —
research use, non-commercial. The bench suite is adapted from `vcruz305`'s (MIT), and SixCat is his.
The numbers are mine.
