# TensorFold (Mia) vs vLLM (Mia) vs myllmbox — same weights, three engines, one gauntlet

2026-10-06 · [@yume_arasaki](https://x.com/yume_arasaki)

[myllmbox.com](https://myllmbox.com/) runs a quality gauntlet across stacks and their TensorFold row
came back broken five times out of five. I have a TensorFold lane and a vLLM lane on the same pair of
Sparks, and I had run [Mia's TensorFold kit](reports/2026-10-02-glm53-flash-mia-tensorfold-two-sparks.md)
and [her vLLM kit](reports/2026-09-09-glm53-flash-exl3-two-sparks.md) before, so I reproduced the test
here. Then I ran the same weights under both engines, and put the myllmbox Qwen lane through the same
battery for a third column.

Short version: the gauntlet failures are real but they are *code* failures the model makes, not
corruption coming out of TensorFold — they happen under vLLM too. Where the engines actually differ is
end-of-turn discipline on short non-English prompts, and there TensorFold's shipped default is the weak
setting, not TensorFold itself. The myllmbox Qwen lane is the strongest of the four on that axis.

---

## What the gauntlet is

Two prompts. `pasture` wants a single self-contained HTML file drawing an animated SVG pasture — sky
gradient, sun with rays, drifting clouds, trees, and exactly four animals with a mandated part list,
each bouncing. `fish` wants the underwater equivalent with exactly two fish that swim and wrap, rising
bubbles, and seaweed. Sampling: thinking on, temperature 1.0, top_p 0.95, one request at a time.
Verdicts are visual — clean, anomaly, broken.

I kept the prompts and the sampling, and wrote two tools so the verdicts are machine-checkable instead
of eyeballed: a generator that runs them against a live serve, and a Chromium harness that renders the
result, collects console and page errors, checks the prompt's hard requirements (no external refs, no
emoji, no scrollbars, size limits), and counts the scene's subjects — pasture animals identified by
their leg rectangles, fish by whether they come on screen and move.

That judge matters, because I got a verdict wrong on the first pass and had to correct it (below).

## The false start you should know about

Two traps, both mine.

**The engine is deterministic per request.** Identical request bytes return byte-identical output even
at temperature 1.0 — temperature and `seed` are both honoured, they just make the *same* request
reproducible. So my first three "samples" were one generation repeated three times, and my first judge
version over-flagged a correct nested animation, which is how I ended up telling myself pasture failed
3 of 3. With distinct seeds and a judge that reads rendered geometry instead of markup, it is 1 of 3 —
in *both* engine configurations.

**The first TensorFold row I looked at was not like-for-like.** Their entry is a Qwen MLX 4-bit
checkpoint on the single-Spark kit; mine is GLM with Mia's own quant on the two-Spark kit. Different
checkpoint, different quant family, different engine version, and they allowed a 70,000-token reply
where my server caps at 32,768.

## Results

**Thinking on, at my 32,768-token cap, produced no file at all.** `pasture` returned 0 characters of
HTML after 687 s: all 32,768 completion tokens were reasoning tokens, still designing geometry when the
budget ran out. The reasoning is coherent — my garble scanner finds no loop or drift in 107k characters
— it just never gets to the code. Their harness allowed 70k, which is the difference between a broken
artifact and no artifact.

**Thinking off, both scenes render, and both engines fail on the same thing.**

| Sample (3 distinct seeds each) | TensorFold `q4` | TensorFold `fp8` | vLLM EXL3 (pre-TF kit) | myllmbox Qwen dual |
|---|---|---|---|---|
| pasture | 1 of 3 broken | 1 of 3 broken | **3 of 3 broken** | 1 of 3 broken |
| fish | 2 of 3 ok | 2 of 3 ok | 3 of 3 ok | 2 of 3 clean |

The pasture failure mode is identical under both engines and I confirmed it by eye: the four animals
collapse into the top-left corner and the field stays empty. In the generated code the bounce is a CSS
keyframe animation on the same element that carries the animal's placement `transform`, and a CSS
transform overrides the SVG transform attribute, so every animal's declared position is thrown away
while the animation runs. Disable the animation and all four are correctly on the grass. The vLLM
samples have the same defect, so it is model behaviour on a fiddly prompt, not an engine signature.

The other mode I caught is a shared timestamp: two `requestAnimationFrame` loops using one `last`
variable, so the seaweed loop eats the elapsed time and the fish loop always computes `dt ≈ 0` and never
swims. Again: model code.

As a control I wrote both scenes myself and put them through the same judge. They pass — animals on the
grass, fish swimming, no console errors — which is what tells me the prompts are passable and the judge
scores a correct scene as correct.

## Where the engines actually differ: end-of-turn

Mia's TensorFold kit ships its own quality check for this: eight short French coding prompts, 48
requests, thinking off; it counts replies that run to `max_tokens` instead of ending their turn and
measures the probability of ending the turn right after the closing code fence. Her README documents the
trade: `DENSE=q4` can lose the end of turn, `fp8` keeps it at ~10% decode cost.

| Stack (same tool, 48 requests) | Replies cut | P(end of turn) mean / min |
|---|---:|---|
| GLM-5.3-Flash EXL3, TensorFold `q4` (shipped default) | 4 / 48 | 0.68 / 0.23 |
| GLM-5.3-Flash EXL3, TensorFold `fp8` | 1 / 48 | 0.82 / 0.25 |
| GLM-5.3-Flash EXL3, **vLLM, identical weights** | 1 / 48 | 0.86 / 0.57 |
| **Qwen3.8-Flash-Next hibrid48, vLLM dual** | **0 / 48** | **0.99 / 0.95** |

Two things fall out. First, the dense-quant setting is a real quality lever and the fix is one knob:
4 of 48 cut replies at `q4`, 1 of 48 at `fp8`, landing exactly on Mia's published fp8 reference. Second,
on identical weights vLLM edges TensorFold at equal settings — same cut count, higher mean, and a much
better worst case, 0.57 against 0.25. Not a rout; a discipline difference on the tail.

The myllmbox lane has no tail at all here: zero of 48 cut, worst task ends its turn 95% of the time.

## The quantization claim, checked in the file

A claim going around: TensorFold does not handle mixed quantization, it "goes one of a kind", so
important parts are forced to be over-quantized. I read the safetensors headers instead of arguing.

The checkpoint is not one quantization. Both engines are serving
`Mia-AiLab/GLM-5.3-Flash-EXL3-4bpw-TensorFold`: **145.6 GiB of routed experts at EXL3 4-bit**, and
**18.0 GiB of everything else at BF16** — attention 11.5 GiB, shared experts 2.0, head 1.2,
embeddings 1.2, vision tower 1.1, dense MLP 1.0. The config says it outright:
`scope: glm53_routed_experts_only`, `head_bits: 16`. The abliterated EXL3 build on the same disk is
byte-for-byte the same layout, and brandonmusic's upstream TR3 carries the identical config.

So the file is multi-precision by design. The claim lands one level up instead: the *engine* takes that
18 GiB of BF16 and re-quantizes it at load — TensorFold to `q4` by default, vLLM to its own dense fp8 —
and attention is the biggest thing in that group. That is exactly what the end-of-turn table is
measuring, and it is a switch, including the position (`DENSE=bf16`) that touches nothing.

## Engine A/B on identical weights

The pre-TensorFold lane I had run before was Mia's vLLM EXL3 kit. Two of its parts were gone: I had
pruned its image, and the TR3 weights it defaults to were deleted from both Sparks by the TensorFold
kit's own upgrade. I re-pulled the image and pointed the kit at the checkpoint still on disk — the same
weights TensorFold was serving — which makes it a clean engine comparison. vLLM loaded them without
complaint: 850k window, four seats, DFlash2 k=7 drafts, 24/24 boot-shape warmup, clean smoke.

Same weights, same French check: 1 of 48 cut, P 0.86 mean, 0.57 worst. TensorFold at `fp8`: 1 of 48,
0.82, 0.25. TensorFold is faster per generation — roughly 30–40% less wall time per gauntlet sample —
and vLLM is better on the tail. On the gauntlet itself, vLLM was *worse*: 3 of 3 pasture samples broken
against TensorFold's 1 of 3, same defect.

## The myllmbox lane, loaded for the comparison

Current main of `myllmbox/qwen38-flash-next-cluster-recipe` (`0dba90f`, 5 Oct) with my standing pins —
port 8007, host 0.0.0.0, `/srv/bilikaz` for models and cache, served name, 1M context via YaRN 4x, and
the Hermes chat patch. Worth noting: that patch is in the tree now as `patches/hermes-chat`, landed at
`4e4747b` from the PR I opened in September, which is a nicer mechanism than the file bind-mount I was
using. The 99 GiB of hibrid48 weights were still on both nodes, so boot was four minutes and no
download: flashinfer GDN prefill, 29 s weight load, 64 seats.

Its artifacts are also bigger — 19–30k characters per pasture sample against GLM's 10–16k — so its
similar wall times (90–126 s) are not a like-for-like speed comparison, just context for the sizes.

## What this does and does not establish

The gauntlet failures under both engines are model code, not engine corruption: no garbling, no token
drift, every static element correct, no console errors on the passing samples. The engine difference
that does show up is a serving-config lever — dense precision — and it is measurable and reversible.

Not established: that TensorFold is the cause of anything in the gauntlet. Same weights under a
different engine reproduced the same failure. The French result is closer to a real engine signal, but
it is one axis, one model, one pair of boxes, and vLLM's edge there is a tail effect on 48 requests.

Numbers are mine, measured on two DGX Sparks. The recipes are Mia's and myllmbox's; the gauntlet is
myllmbox's idea, run here with my own generator and judge.
