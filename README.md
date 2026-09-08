# yume_benchmarks

Receipts from [@yume_arasaki](https://x.com/yume_arasaki). What we measured on our desks. Not a harness, not a recipe fork, not a how-to.

| Date | Report |
|---|---|
| 2026-09-09 | [GLM-5.3-Flash EXL3 on two DGX Sparks](reports/2026-09-09-glm53-flash-exl3-two-sparks.md) |

New runs become a new file under `reports/`. Dated. One model, one kit, one write-up.

## 2026-09-09 in one screen

Two GB10 Sparks, Mia’s EXL3 dual recipe `@9c0794b`, thinking off, decode after first token.

| | |
|---|---|
| Count 1→200 | **66.4 / 133.4 / 187.6** at 1 / 2 / 4 streams |
| Essay / code / JSON | **28.5 / 61.3 / 44.7** |
| Needles 8k→256k | **12/12** |
| Decode vs depth | **21.6 → 24.7** (no fall-off) |
| Warm session, build a tiny game | **43.8** tok/s at ~34k prompt tokens |

Her public structured bar is 62.9. Ours is 66.4. We are not printing a delta.

License: MIT. The write-ups are ours. The serve recipe is Mia’s.