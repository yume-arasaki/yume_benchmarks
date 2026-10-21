# DeepSeek-V4.1-Flash-1017 on two DGX Sparks — Mia's TensorFold kit, and a pin that no longer exists (2026-10-21)

2026-10-21 · [@yume_arasaki](https://x.com/yume_arasaki)

A new checkpoint (1017) on a **new engine**: not vLLM this time, but TensorFold's native Zig server
from BertholomausAI's `dsv41` family, TP=2 across the pair. Mia's recipe drives it with one script,
puts Engram's tables on local disk where they belong, and publishes numbers I could check against
mine. So I ran it end to end, gridded it with my own agent battery, and pushed the window.

Two things came out of it. The performance is real — her C1 figures reproduce to within 2%, the
1M window genuinely serves a 1,039,888-token prompt, and this is the first lane on this pair that
passes the maths clock without a scratchpad. And **the recipe cannot currently be installed from its
own pin**: the checkpoint repo was force-squashed, the pinned commit is gone, and it never contained
the `tokenizer.json` the prepare step needs. That half of the report is a receipts list for anyone
who tries.

**Strain:** `dsv41-1017-flash-exl3/tensorfold-tp2/mia/6d54be4f`
**Recipe:** [MiaAI-Lab/DeepSeek-v4.1-Flash-1017-2x-DGX-Sparks](https://github.com/MiaAI-Lab/DeepSeek-v4.1-Flash-1017-2x-DGX-Sparks) `@75cab1d`
**Checkpoint:** `Mia-AiLab/DeepSeek-V4.1-Flash-1017-EXL3-EAQ-v1-2.9-bpw` @ `6d54be4f` — experimental quant (see correction below)
**Engram:** `deepseek-ai/DeepSeek-V4.1-Flash-1017` @ `1bae0e10`, shards 47 and 48 only, on local disk on both nodes
**Engine:** TensorFold `deepseek-v41-zig` @ `629c524a`, Zig 0.17.0, image `tensorfold-dsv41:exl3-tp2-8eb5919ff0a4` (built locally — no GHCR tag for these pins)
**Serve:** spark-1 **:8007**, TP=2 over RoCE, served id `DeepSeek-V4.1-Flash-1017`, `max_batch_size` 4
**Protocols:** Mia's `tools/bench.py` (thinking off, T=0, fixed seed) and the house agent grid (same pins), kept apart from her sparkDash figures (drafts on, thinking default)

## Headline

| | |
|---|---|
| Her C1 figures, reproduced | prose **73.5** vs 74.9 · code **110.6** vs 109.9 |
| Her own `bench.py`, run as-is | **58.2** at one stream, **119.2** at four (her bands 55–61 / 113–117) |
| Her C2/C4 figures | exceeded: code **183.0 / 186.7** vs 139.8 / 155.5 |
| Prefill, 8K → 256K | **2,004 → 1,926 tok/s** (hers 2,166 → 2,000) |
| Needle | **3/3 at 513,819** (hers 3/3 at 479,012) |
| Concurrency | peak **119.0 tok/s at C4** (the batch size), then 68.8 at C128 with **128/128 completing** |
| The maths clock | **passes**, `exact_match: 1` thinking off *and* on — first lane here to do it |
| The window | 524,288 gridded; **1,048,576 loads**, and a **1,039,888-token prompt** answers 3/3 in 669.9 s |
| Garble | 0 in 12 night-1 cells; one real spec violation (`fence` at `code@32k`) |

## Installing it is the story

Four hours of this run went into things the README cannot tell you, because they changed after it was
written. In order:

**1. The pinned commit does not exist any more.** `scripts/config.sh` pins the checkpoint at
`6d54be4fca338d79b17c3b03cc9a423bb3ac6718`. The Hub repo was force-rewritten on 2026-10-19
(`Super-squash branch 'main'`), and that commit now answers `Invalid rev id`. My 197 GiB came down
before the climb was collected; anyone starting today gets a different revision, or none.

**2. That commit never had a tokenizer.** The prepare step builds Engram's token map from the
checkpoint's `tokenizer.json`. The snapshot had 44 files and none of them a tokenizer — the file
exists on `main` now, which is exactly what made this confusing: the Hub looks complete, the pin
wasn't. Prepare died after the 400 GiB of downloads with
`Exception: No such file or directory (os error 2)`.

The fix is to take the tokenizer from the **base** model at its own pin and put it in the checkpoint
snapshot as a **real file** — a symlink dangles, because the container only has `/hf` mounted and
HF's own snapshot links are relative for that reason:

```bash
curl -L -H "Authorization: Bearer $HF_TOKEN" -o "$SNAP_C/tokenizer.json" \
  "https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash-1017/resolve/1bae0e10/tokenizer.json"
```

Prepare then reports the check that matters: *"Token map: the same bytes the kit's Engram fixtures
were recorded with."* That is not "a tokenizer loaded" — it is the right one.

**3. The head has to be an NFS server, and the documented export is not enough here.** The README's
line is `echo "$HOME/.cache/huggingface <worker>(ro,no_subtree_check)"`, which implies the default
`root_squash`. On this desk `~` is `750` and `.cache` is `700`, so the worker's container root
squashes to `nobody` and `find /hf` returns **Permission denied**. Export with `no_root_squash` on a
read-only export. And the head only had `nfs-common` — it needed `nfs-kernel-server` installed.

**4. `hf` must be on the non-interactive PATH.** It was at `~/.local/bin/hf`
(huggingface_hub 1.29.0), which the kit's subshells do not see; symlinking it into `/usr/local/bin`
fixed it. The worker does not need it.

None of that is Mia's error — the repo moved under the recipe. It is the difference between "a recipe
you can read" and "a recipe you can run", and it took the run from one command to a scavenger hunt.

## Her numbers beside mine

Her published figures come from `sparkDash` with drafting on and the recipe's sampling (T=1.0,
top_p 0.95, top_k 20, thinking on). Mine come from the house grid at thinking off, T=0. Where the
lanes are the same clock — prose-600, code clamp — the comparison is direct; where they are not, I
say so.

| Measurement | Hers | Mine | Delta |
|---|---:|---:|---:|
| prose-600, 1 / 2 / 4 streams | 74.9 / 105.5 / 130.0 | **73.5 / 103.2 / 140.5** | −2% / −2% / +8% |
| code clamp, 1 / 2 / 4 streams | 109.9 / 139.8 / 155.5 | **110.6 / 183.0 / 186.7** | +1% / +31% / +20% |
| prefill, 8K → 256K | 2,166 → 2,000 | **2,004 → 1,926** | −8% / −4% |
| needle, three placements | 3/3 at 479,012 | **3/3 at 513,819** | deeper |
| concurrency | ~130–155 at 4 streams | **119.0 at C4** | −8 to −23% |
| `tools/bench.py`, 1 stream | 55–61 | **58.2** (57.7–62.0) | inside |
| `tools/bench.py`, 4 streams | 113–117 | **119.2** (109.8–120.0) | +2% |

The C1 numbers are the fair test and they land within 2%. The C2/C4 excess is almost certainly the
prompt set: code clamp is fifty identical one-line functions, which batches beautifully, while her
sparkDash code lane is a longer generation. I am not claiming to be faster than her; I am reporting
that on these two clocks the shapes differ above C1 and saying why.

**Her `bench.py` removes that doubt.** It is her script, her prompts, her seed — so the only
remaining differences are the port and the thinking pin. I ran it as shipped:

| `tools/bench.py` | 1 stream | 2 streams | 4 streams |
|---|---:|---:|---:|
| decode tok/s, median | **58.2** (62.0 / 58.2 / 57.7) | **43.9** per stream | **31.3** per stream |
| aggregate tok/s | 61.1 (63.1 / 61.1 / 59.5) | 86.1 | **119.2** (119.2 / 120.0 / 109.8) |
| prefill, 2K / 8K / 32K | 1,881 / 2,188 / 1,878 | | |
| needle | 3/3 at 7,713, 30,696, 122,631 · 0 errors | | |

One stream lands in the middle of her 55–61 band; four lands two points above her 113–117 band. The
"we are faster at C4" claim evaporates under her own harness — the house code-clamp prompt was the
reason, exactly as suspected. **The one figure of hers I cannot reproduce is drafting.** The server
reports `drafts: true` in every response, but exposes no accepted/round counter, so her 2.2–2.4
tokens a round is visible only as an effect on decode rate, not measurable here.

**The maths clock is the surprise.** `2^10 + 3^5` with "output only the integer" has produced a
wrong deterministic integer on GLM-5.3, DSV4.1, two 27B quants, MiMo and hibrid48. This lane returns
**1267, `exact_match: 1`, on both the thinking-off and thinking-on clocks.** It is the first lane on
this pair to pass without a scratchpad, and it is the kind of result that suggests the newer
checkpoint handles exponent arithmetic in a single pass rather than approximating.

## The grid at 524,288

Thirteen isolates. Night 1 (wall-clock aggregate, mean of two reps):

| Lane | C1 | C2 | C4 |
|---|---:|---:|---:|
| structured count-200 | 130.3 | 211.7 | 224.2 |
| prose-600 | 73.5 | 103.2 | 140.5 |
| code clamp | 110.6 | 183.0 | 186.7 |
| json catalog | 54.4 | 80.4 | 95.6 |

Garble **0** in all twelve cells. Tools: real `tool_calls`, no XML.

The ladder, three placements a depth:

| Depth | 8,192 | 32,768 | 131,072 | 262,144 | 524,288 | 900,000 |
|---|---|---|---|---|---|---|
| needle | 3/3 | 3/3 | 3/3 | 3/3 | **3/3** | refused |
| prefill tok/s | 2,004 | 2,054 | 2,019 | 1,926 | 1,766 | — |
| TTFT | 4.1 s | 15.6 s | 63.6 s | 133.3 s | 291.0 s | — |

Decode at depth holds 60–70 tok/s from 8k to 512k. Work at depth (32k / 131k): structured 133.8 /
130.4, prose 73.3 / 70.9, code 118.6 / 117.7, tools real at both.

**One genuine garble, and it is a spec violation.** The `code@32768` job wrapped its answer in a
`` ```python `` fence although that clock's prompt says *"no comments, no fences, no extra text"*.
The detector flagged `fence` and is right to. At 131k the same cell came back clean — a
depth-dependent instruction lapse, not corruption.

The 35k towers came back clean, including `hermes@35k` with a real `tool_calls` array. Worth noting
against the RED-SNOW lane on this same pair, where the OMP tower produced XML in content in one of
two passes: here it did not.

## Concurrency: it batches, then it queues

| Streams | 1 | 2 | **4** | 8 | 16 | 32 | 64 | 128 |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| wall aggregate | 55.9 | 81.2 | **119.0** | 86.4 | 77.2 | 73.0 | 69.1 | 68.8 |
| per-stream | 74.2 | 53.2 | 37.0 | 35.3 | 35.5 | 35.2 | 34.3 | 34.5 |
| streams ok | 1/1 | 2/2 | 4/4 | 8/8 | 16/16 | 32/32 | 64/64 | **128/128** |
| TTFT median | 1.3 s | 1.6 s | 1.6 s | 9.9 s | 27.3 s | 61.5 s | 133.9 s | 272.4 s |
| J/completion-token | 1.90 | 1.37 | **1.00** | 1.04 | 1.04 | 1.05 | 1.07 | 1.07 |

This is the opposite shape to RED-SNOW on the same hardware. There, throughput was flat from one
stream to 128 — a queue. Here it climbs to a peak exactly at the **batch size of four** (119.0 tok/s,
and the best energy point at 1.00 J/token), then *declines* to ~69 while every stream still
completes. Past four, extra streams buy queue position rather than throughput, but the engine never
sheds them.

## The window

`CONTEXT=1048576` loads, and I did not expect it to: Mia's README records a 655,360 run killed by
memguard on a head with other software, and this head runs sparkDash plus a container set. It came up
in 26.6 s with 6.9 GiB free on the head and 9.3 GiB on the worker, above the 2.5 GiB floor. `dmesg`
logged ten `NV_ERR_NO_MEMORY` lines during the restart — the old container's memory was still held
while the new one allocated — and it recovered.

A prompt near the window is accepted, so this is a real window and not a banner:

```
accepted: prompt_tokens=1039888 completion_tokens=107 elapsed=669.9s
prefill_rate_report=1552 tok/s (prompt + decode, wall clock)
placements P05/P50/P95: {'KIWI-9472-EMBER': True, 'PLUM-3310-TIDE': True, 'ONYX-8065-VALE': True}
MAXCTX_PROMPT_OK prompt_tokens=1039888 needles=3/3
```

1,039,888 tokens, three phrases planted at 5/50/95% of the span, all returned in order, eleven
minutes wall clock.

**The caveat is in the engine's own line:** *"model 104.96 GiB; 4 streams at once, 0.00 GiB each at a
1048576-token window, of 0.0 GiB left after a 12.2 GiB reserve."* The window is offered with no
static KV budget per stream; one long prompt works because the cache grows as it goes. Four
concurrent long prompts are untested and, on that arithmetic, unlikely to fit. **Treat 1M as one
long conversation, not four.** That arithmetic also explains the contradiction with her README: what
gets allocated at load is the weights, not the window, so a bigger reserved window need not blow up
the load — it blows up when you fill it four times over.

## The grid at 1,048,576

Thirteen more isolates under the `8007-ctx1m` label: same prompts, same seeds, only the window and
the run tag differ.

| Lane (wall aggregate) | C1 | C2 | C4 |
|---|---:|---:|---:|
| structured count-200 | 130.9 | 215.3 | 248.3 |
| prose-600 | 71.9 | 108.2 | 140.1 |
| code clamp | 112.1 | 181.3 | 179.1 |
| json catalog | 54.0 | 83.3 | 95.0 |

Garble **0 across 60 streams**. The maths clock passes again, both thinking pins. The 35k towers
come back clean — `omp@35k` 110.4 tok/s, `hermes@35k` 95.5, both with real `tool_calls` and no XML
in content.

The ladder now runs to where the window actually ends:

| Depth | 8,074 | 32,116 | 128,470 | 256,872 | 513,808 | **882,065** |
|---|---|---|---|---|---|---|
| needle | 3/3 | 3/3 | 3/3 | 3/3 | 3/3 | **3/3** |
| prefill tok/s | 2,037 | 2,128 | 2,049 | 1,948 | 1,778 | **1,590** |
| TTFT | 4.0 s | 15.1 s | 62.7 s | 131.9 s | 288.9 s | **554.7 s** |

Decode at depth: 69.8 / 70.2 / 68.3 / 67.0 / 65.2 / 68.5 tok/s at 8k → 900k. It does not degrade
with history; the 900k rung read slightly *above* the 524k one, which is rung-to-rung noise rather
than a trend.

The sweep is the same shape at the larger window:

| Streams | 1 | 2 | **4** | 8 | 16 | 32 | 64 | 128 |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| wall aggregate | 55.5 | 85.4 | **119.8** | 87.1 | 75.5 | 72.7 | 70.6 | 69.2 |
| per-stream | 73.0 | 53.5 | 37.0 | 35.6 | 35.1 | 35.5 | 35.0 | 34.7 |
| streams ok | 1/1 | 2/2 | 4/4 | 8/8 | 16/16 | 32/32 | 64/64 | **128/128** |
| TTFT median | 1.3 s | 1.4 s | 1.6 s | 9.5 s | 27.2 s | 61.7 s | 132.1 s | 271.3 s |

Against the 524,288 sweep (55.9 / 81.2 / **119.0** / 86.4 / 77.2 / 73.0 / 69.1 / 68.8), every rung
is within noise of the smaller window. **A bigger reserved window costs nothing at these prompt
lengths** — which is consistent with the engine's own arithmetic: the load reserves weights, and the
KV is paid as it is filled. What you pay for is the fill: 555 s of prefill on the 882k needle, and
669.9 s on the 1,039,888-token prompt.

**The one garble repeats exactly.** `code@32768` in job-at-depth wrapped its answer in a
`` ```python `` fence at the 1M window too — same cell, same depth, same prompt, both windows, while
`code@131072` is clean in both. That is a reproducible instruction lapse at one depth, not sampling
noise, and it is the only spec violation in 26 isolates.

## What it cost me

Four hours of install archaeology, described above, most of it after the downloads were done.

A 400 GiB prepare over the head's WiFi: 196 GiB checkpoint at ~26 MiB/s and 189 GiB of Engram at
~63 MiB/s once both shards were in flight. The wired uplink is dead on this pair — six seconds of
`tcpdump` on it caught nothing — so there is no faster path without hardware work.

And one genuinely new failure mode: at 906k on a 524,288 window the serve **accepts the prompt and
streams nothing**, rather than returning a 4xx. My runner crashed on it the first time
(`ttft_s=None`). It now records that as a refusal row. If you write a harness against this engine,
budget for a window refusal that looks like an empty successful response.

## What I did not prove

1. **The quant's quality, at depth.** [Correction 2026-10-21: this report first said "no ppl/KL
   exists". The model card — updated 2026-10-20, after our bench — now carries teacher-forced
   next-token fidelity against the original `1bae0e10` weights: 100 conversations of at most 4,029
   tokens, KL 0.0137 nats [0.0115, 0.0165], top-1 agreement 96.6% [96.2, 96.9], reference top-1 in
   the quant's top-5 on 99.98% of positions, and a paired comparison vs LibertAIDAI's 2.9 bpw build:
   KL −0.00171 nats, top-1 +0.36 pp — EAQ slightly ahead at identical widths. What still does not
   exist: long-context fidelity, perplexity, standard eval scores. So speed numbers remain "this
   quant, on this pair", and the fidelity story is the card's short-context one.]
2. **Four concurrent long prompts.** See the window section. One works; four are unmeasured and the
   engine's own arithmetic is discouraging.
3. **Drafting's accept rate.** `drafts: true` in every response, but no accepted/round counter is
   exposed, so her 2.2–2.4 tokens a round is visible only in the decode rate. If you want that
   number, it needs the server to report it.
4. **Anything past 524,288 under load.** The 1M sweep is the same short prompts as the 524,288 one;
   it does not fill the window.
5. **The abliterated overlay.** `ABLIT=1` is a gated repo and a separate experiment; it stayed off.

## Isolates

- `2026-10-21_8007_*` — 13 isolates at the 524,288 window
- `2026-10-21_8007-ctx1m_*` — the same three phases at the 1,048,576 window
- Mia's own: `~/src/DeepSeek-v4.1-Flash-1017-2x-DGX-Sparks/runs/`

## Credit

Recipe, engine fork and quant: [MiaAI-Lab](https://github.com/MiaAI-Lab), with TensorFold by
BertholomausAI (`dsv41` family, Zig), and the RoCE/kernel work credited in her README. Weights:
DeepSeek's V4.1-Flash-1017, quantized by Mia; Engram tables from the base release. The numbers are
mine, and the install fixes above are this desk's.
