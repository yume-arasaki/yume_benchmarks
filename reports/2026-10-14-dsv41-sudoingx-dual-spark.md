# DeepSeek V4.1 Flash on dual DGX Sparks — sudoingX build (2026-10-14)

Fourth DeepSeek strain, fourteenth agent-grid. First run of the sudoingX recipe (v3: EXL3 2.7 bpw
+ own decode kernels + DSpark k5, vLLM 6e517b15 + 20 patches, TP=2 over the fabric on our port-1
layout, GID 5 both nodes). Serve: `:8007`, 262,144 ctx (model native 1M, launcher caps), MAX_SEQS=2,
thinking-off via the documented `chat_template_kwargs.enable_thinking` knob.

**Strain:** `dsv41-sudoingx-exl3/vllm-tp2/sudoingx/v3-dk5`
**Recipe:** github.com/sudoingX/dgx-spark-deepseek-v41
**Protocol:** §21 family (v2 streaming + non-streaming wall companions + burst guard + mixture transcripts).
The full artifact trail — five catches in one night — is the story of this report.

## Headline numbers

| Lane | tok/s | Protocol |
|---|---|---|
| structured count-200 | 135.5 | streaming, 3 reps, spread 0.2% |
| code clamp | 91.3 | streaming, 3 reps, spread 5.1% |
| json catalog | 115.2 | streaming, 3 reps, spread 1.9% |
| maths numeric | 114.0 | streaming, 3 reps, spread 0.4% |
| prose 400w | 57.2 | streaming, 3 reps, spread 4.5% |
| ladder 8k→240k | 57.6–61.5 FLAT | streaming, 200 tok rungs, salted |
| needle 8k/65k/240k | 6/6 exact strings | both 64- and 512-token budgets |
| OMP tower (mixture) | 108.3 wall-bound | wall companion authority |
| Hermes tower (mixture) | 88.2 wall-bound | wall companion authority |
| TTFT @35k | 17.6–17.8 s | ladder-consistent prefill |
| prefill implied | 1,404–1,893 tok/s | pt/TTFT across ladder |

## The five artifact catches (methodology §21.1–21.5)

1. **Chunk-burst trap (21.1).** First grid read 455–1,031 tok/s on structured/json. Cause: this
   engine flushes one SSE chunk per accepted DSpark step (~7.7 tok/chunk at acceptance 2–5), so
   content-window division inflates by tokens-per-chunk. Caught by physics bound; confirmed by
   server-side `Accepted throughput` metrics landing in the honest band. Burst guard
   (`span < ct/200` flags) added to every streamed row.
2. **Reasoning-key parser (21.2).** vLLM's deepseek_v41 parser emits `delta.reasoning`; the GLM
   convention is `reasoning_content`. Content-only parser read code/prose lanes as zero content
   (model was burning budgets thinking). Zero content chunks = parser bug until proven otherwise.
   Thinking-off knob set explicitly for speed lanes.
3. **Tool-args-as-workload (21.3).** DSV41 streams real work as tool-call arguments (OMP: the
   whole game inside `write_file` args, 5–6.5K chars vs 66 chars prose). Content-window timing
   read 400–17,000 tok/s. Honest agent-turn window = all deltas + captured tool payloads.
4. **Token calibration (21.4).** Char-ratio packing tuned on RED-SNOW's tokenizer landed 23.5k
   on DSV41 — a 30% depth miss. All client-lane prompts now `/tokenize`-calibrated; v3 landed
   34,672–34,679.
5. **Buffered-flush padding (21.5, the big one).** Streaming all-delta window read 175 tok/s;
   the non-streaming wall companion read 2048/(35.3 − 17.7) = ~116. ~6 s of real generation hid
   in pre-first/post-last buffer padding. Streaming number retired; wall companion is the
   authority for turn-shaped workloads. Joey's "that number doesn't sit right" beat four
   deterministic gates.

## Mixture-realistic towers (21.6 + v4)

Simulated mid-loop harness transcripts (system policy, prior turns, tool calls WITH results)
instead of cold single turns. OMP 106–110 wall-bound (model still goes tool-pure: file-dump
style); Hermes 82–94 with genuine interleaved narration + dual `web_search` calls, valid JSON
args every rep. Tower÷prose ratio: Hermes 1.5× — inside/near the 0.8–1.4× reference band every
Grok-era qwen38 strain occupied. OMP 1.9× = drafter-perfect case (server acceptance 4.6–5.4 on
k=5), printed as ceiling not expectation. Forecast for real loops: 70–100 tok/s at 35k.

## Tool calling

Verified payloads: dual parallel `web_search` calls, valid JSON args (schema-exact `query` key),
distinct IDs, sensible task decomposition — in every mixture rep. finish_reason `tool_calls`
clean. The inverse of the RED-SNOW finding (tools attached, never called, XML in content).

## Caveats

- MAX_SEQS=2; 262k cap (native 1M; launcher default, KV trade), API-enforced.
- OMP `write_file` args JSON-truncate at the 2048 cap mid-file — full delivery needs higher
  max_tokens on this tokenizer; NOT malformed calling.
- Energy rail not measured tonight (dash carries class estimate, labeled).
- v4 mixture prompts landed ~24.5–25.1k (history-chunk granularity), stamped in-isolate.
- Retired numbers (never reuse): 455/1031 burst reads, 175/16859 stream windows, 46.9/48.5
  chassis values, "around 2,000 tok/s prefill."

## Isolates

- `2026-10-14_8007_dsv41-grid-stream` — 5 lanes × 3 reps + wall companions
- `2026-10-14_8007_dsv41-ctx-ladder` — 7 depths, flat decode, TTFT line
- `2026-10-14_8007_dsv41-needle` — 6/6, both budgets, think-off
- `2026-10-14_8007_dsv41-towers-35k` — v3 token-calibrated (SUSPECT stamp on stream numbers,
  wall-companion supersession receipt `omp_wall_bound.json`)
- `2026-10-14_8007_dsv41-towers-mixture35k` — v4 mixture transcripts, the shippable tower pair
- `2026-10-14_8007_dsv41-towers` — v1/v2 superseded trail (kept for the artifact record)

## Context

Mia's EXL3 2-node kit (strain `dsv41-flash-exl3/vllm-tp2/mia/8530568`, 2026-09-14) read 30–35
at depth on the same boxes; this build holds 57–61 on the same lanes — 1.8–2× in five weeks.
Her separate 3–4× SGLang line (TP4: 87.7 prose, 1M ctx needle pass) is a different topology.
sudoingX's own baseline runs: TensorFold v0.6.0 engine (Zig `ce20a54`) with Mia-AiLab's 2.9 bpw
pack, executed by them on the same boxes and client — this build wins code +2.5%, structured
+1.7%, loses prose 58.3 vs 62.1 and prefill. Note: TensorFold is a serving engine, not a
dual-Spark DSV41 recipe; the baseline was their run, not a separate product.

DeepSeek reference implementation).
