# Qwen3.8-Flash-Next on one DGX Spark (bilikaz v5.1, full grid)

2026-10-03 · [@yume_arasaki](https://x.com/yume_arasaki)

One Spark, no cable, no second box. [vr8vr8's single-Spark recipe](https://github.com/myllmbox/qwen38-flash-next-recipe) for Qwen3.8-Flash-Next hit v5.1 this week, and I ran the full agent grid on it. The interesting question was simple: what does half the hardware actually cost you. I have the dual-Spark grids on the same weights family to answer it with.

Strain: `qwen38-fn-hibrid48-solo/vllm-tp1/bilikaz/79223f6`. vLLM 0.30 in his image (pinned by digest), RecoverSSM, dynamic draft depth 3 to 7, a 876k-token KV pool, 16 seats, 262k context window. Port 8007, one GB10. Same frozen clocks as every grid this desk runs, thinking off, temperature 0, unique salt per stream.

One personal note before the numbers. The recipe ships an optional patch called `hermes-chat`. I wrote that patch. My dual-Spark grid on his earlier recipe hit harness friction — thinking defaulting on, temperature handling — I contributed the fix upstream, and he shipped it in this kit with credit. First time I have deployed my own patch inside someone else's recipe. It works.

---

## The setup

Deploy is three commands and one patience test: 98 GB of weights, about ninety minutes, then a table map builds itself on first boot and the serve comes up healthy in about four minutes. The recipe pins its image by digest and its weights by repo, so nothing moves under you between boots.

Thinking is on by default (model native). `chat_template_kwargs: {enable_thinking: false}` works clean, and the hermes-chat patch maps my agent harness's reasoning object onto it, both directions. Tool calls parse exactly — the XML-in-content failure mode from the early dual runs is gone.

## Empty context

Two runs a lane, headline is the better one. Decode aggregate after first token.

| What I asked | 1 stream | 2 streams | 4 streams |
|---|---:|---:|---:|
| Count from 1 to 200 | 81.7 | 146.2 | 244.4 |
| Explain a hash map (600 tok) | 58.2 | 89.6 | 134.3 |
| Fifty identical Python clamps | 83.9 | 132.6 | 236.6 |
| A JSON blob of fake GPU stats | 69.3 | 111.9 | 207.1 |

## What half the hardware costs

My dual-Spark grid on the same weights family, same clocks ([28 Sep report](reports/2026-09-28-qwen38-flash-next-hibrid48-v41-two-sparks.md), strain `8ec444f`):

| Lane, 1 stream | 1× Spark | 2× Spark | solo/dual |
|---|---:|---:|---:|
| Count to 200 | 81.7 | 131.8 | 62% |
| Hash map explainer | 58.2 | 78.4 | 74% |
| Fifty Python clamps | 83.9 | 126.4 | 66% |
| JSON GPU stats | 69.3 | 111.8 | 62% |

Roughly 60 to 70 percent of the dual numbers, lane depending. At four streams the gap narrows to 66–78 percent. One confound printed with the table: the dual grid ran his v4-class recipe (fixed K=5), this one his v5.1 (RecoverSSM, dynamic depth). Some of the closeness is the engine improving, not the hardware. Same weights family, same clocks, different recipe vintage — I print the ratio with that label attached.

## Sixteen seats

The thing the single box does that the dual never did on this recipe: concurrency scale. Prose-600 aggregate ladder:

| Streams | Aggregate |
|---|---:|
| 1 | 50.7 |
| 2 | 80.7 |
| 4 | 122.9 |
| 8 | 189.6 |
| 16 | 282.6 |

My dual serve on this model was measured to four streams — 383.1 aggregate structured at C4 — with the recipe configured well past that. Sixteen concurrent completions on one desk-side box. If your work is many parallel agents rather than one deep one, this shape wins.

What RecoverSSM changed: this model mixes attention layers with recurrent state-machine layers, and a speculative draft used to mean saving and restoring the recurrent state for every drafted branch. His port verifies all drafts from one saved state and replays only the accepted tokens. Deeper drafts stop costing memory, which is what lets depth go to 7 while the KV pool grows to 876k tokens on the same silicon. His numbers say it plainly: the pool was 813k at depth 6 on v5, now 876k at depth 7.

## The context window

262k native, no YaRN stretch. Decode after packed filler, 256 forced tokens:

| Context | Decode |
|---|---:|
| 8k | 48.6 |
| 32k | 48.4 |
| 131k | 50.2 |
| 262k | 49.4 |

Flat, wall to wall. The box holds its entire window without sagging. The dual grid's YaRN stretch to 1M still belongs to the dual — its ladder read 64.1–66.2 across six rungs including 900k — but within 262k the single Spark does not bend.

Needle at four depths, three positions each: 12 of 12 found, deepest rung 255,787 actual tokens.

## Agent lanes at 35k

The OMP tower writes 2,048 tokens of a real single-file HTML game at 77.2 tok/s after a clean tool call. The research client lands a search-then-build turn at 57.1. The generate follow-ups, 2,048 streamed tokens with no tools after the round-trip: 77.3 and 80.7. Tool parsing exact on every lane: proper tool_calls objects, no XML drool.

For reference, the dual read 74.7 on the same research lane — the single box holds 76 percent of it. The dual's OMP row (183.3) is a measurement I flagged as suspect on my own 26 Sep dashboard (a 35-token turn timed over a 0.2-second window), so I do not chart against it; the honest statement is "dual faster on the build lane, magnitude uncertain."

Dynamic draft depth is the quiet star here. Prose settles shallow, code drafts deep, and the engine moves each request's depth by measured acceptance. It is the same insight as my accept-rate tables, automated: stop paying for draft positions nobody accepts.

## What it costs

Energy at the C16 ceiling: 0.20 J/tok on my summed GPU rail — and that figure still includes the second Spark idling, so the true single-box number is lower. Cheapest per-token figure this desk has printed. The tradeoffs: roughly 60 to 70 percent of dual decode at one stream, a 262k ceiling instead of 1M, and the second Spark sits idle unless you run something else on it.

Which is the actual conclusion. This recipe makes the second Spark optional, not wasted. One box runs the agent fleet at 16 seats while holding its whole window flat. The other box is free for a GLM, a DeepSeek, whatever the week needs. Two recipes from the same builder now cover both shapes: his dual for one deep 1M-context session, his single for a fleet of parallel ones. Fleet topology as a menu, not a marriage.

## Gotchas from the boot

- No `/v1/tokenize` endpoint on this build. My pack calibrator fell back to a hardcoded constant until I forked it to live-completion calibration. If you bench this, size prompts by `usage.prompt_tokens`, never by character count.
- The 262k window is the window. My depth ladder tried 524k and the serve correctly refused it with a 400.
- A two-pass pack refine warms the prefix cache — prefill numbers on the refine pass read as cache hits. Decode and retrieval cells stay valid.
- The arithmetic probe stays in the video appendix. Eighth model, same cell, same quirk. It is the probe, not the models.

## Credit

Recipe: [myllmbox/qwen38-flash-next-recipe](https://github.com/myllmbox/qwen38-flash-next-recipe) `@79223f6`, image `myllmbox/qwen38-flash-next-vllm:v5.1` by vr8vr8 (bilikaz on GitHub, myllmbox on HF). Weights: [myllmbox/Qwen3.8-Flash-Next-hibrid48](https://huggingface.co/myllmbox/Qwen3.8-Flash-Next-hibrid48). RecoverSSM: vllm-project/vllm#58863 by jschmied, ported to 0.30 by the recipe author. The hermes-chat patch: mine, [PR #2](https://github.com/myllmbox/qwen38-flash-next-recipe/pull/2). Upstream model by Qwen. The numbers are mine.
