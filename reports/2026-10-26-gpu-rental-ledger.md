# GPU rental ledger — global market, 2026

Receipts for the rental post (`2026-10-26`, "a16z said GPUs age like fine wine"). Every number printed in the post maps to a row in this ledger. Tiers and rules are at the bottom.

Raw data: [`ledger.jsonl`](../data/gpu-rentals/ledger.jsonl) — 246 observations, one tuple per row.

Chart: [`assets/gpu-rental-matrix-2026-10.png`](../assets/gpu-rental-matrix-2026-10.png)

## Headline numbers, and where each one comes from

| Number in the post | Value | Source | Tier | Window |
|---|---|---|---|---|
| H200 rental, YTD | +160% | Ornn Data OCPI, as printed by a16z Charts of the Week | R1 transacted | 1 Jan → 23 Oct 2026 |
| B200 rental, YTD | +120% | same | R1 transacted | 1 Jan → 23 Oct 2026 |
| RTX PRO 6000 rental, YTD | +65% | same | R1 transacted | 1 Jan → 23 Oct 2026 |
| H100 rental, YTD | +35% | same | R1 transacted | 1 Jan → 23 Oct 2026 |
| H100 listed, YTD | +43.1% | GPU Economy `historical_series` via SkyPilot catalog | R2 listed | wk 5 Jan → wk 19 Oct 2026 |
| H100 SXM transacted | $3.03/hr | Ornn `/api/daily-index` | R1 | settled 25 Oct 2026 |
| H200 transacted | $6.30/hr | same | R1 | 25 Oct 2026 |
| B200 transacted | $9.70/hr | same | R1 | 25 Oct 2026 |
| A100 SXM4 transacted | $1.09/hr | same | R1 | 25 Oct 2026 |
| RTX PRO 6000 WS transacted | $1.53/hr | same | R1 | 25 Oct 2026 |
| H100 listed median | $3.54, n=41 providers | GPU Economy spread page | R2 | 26 Oct 2026 04:40 UTC |
| H200 listed median | $4.56, n=54 | same | R2 | 26 Oct 2026 |
| B200 listed median | $6.99, n=38 | same | R2 | 26 Oct 2026 |
| A100 80GB listed median | $2.00, n=62 | same | R2 | 26 Oct 2026 |
| RTX PRO 6000 listed median | $2.32, n=40 | same | R2 | 26 Oct 2026 |
| H100 cheapest / dearest | $1.79 Cudo / $11.06 Azure | same | R2 tails | 26 Oct 2026 |
| H200 cheapest | $2.09 Beam | same | R2 tail | 26 Oct 2026 |
| B200 cheapest | $4.11 | same | R2 tail | 26 Oct 2026 |
| H200 spot median | $2.97 | GPU Economy `prices.json` (market=spot) | R3 | 26 Oct 2026 |
| H200 committed median | $2.79 | GPU Economy `prices.json` (market=reserved) | R4 | 26 Oct 2026 |
| RTX 3090 listed median | $0.50, n=9 | GPU Economy provider page | R2 | 26 Oct 2026 |
| RTX 4090 listed median | $0.47, n=16 | same | R2 | 26 Oct 2026 |
| RTX 5090 listed median | $0.84, n=19 | same | R2 | 26 Oct 2026 |
| RTX A6000 listed median | $0.57, n=20 | GPU Economy spread page | R2 | 26 Oct 2026 |
| RTX 6000 Ada listed median | $1.06, n=11 | GPU Economy provider page | R2 | 26 Oct 2026 |
| 3090 payback | 2,412 h → 7 mo @50% | $1,206 sold ÷ $0.50/hr | computed | 26 Oct 2026 |
| 4090 payback | 4,711 h → 13 mo | $2,214 sold ÷ $0.47/hr | computed | 26 Oct 2026 |
| 5090 payback | 5,765 h → 16 mo | $4,843 sold ÷ $0.84/hr | computed | 26 Oct 2026 |
| PRO 6000 payback | 6,207 h → 17 mo | $14,400 ÷ $2.32/hr | computed | 26 Oct 2026 |
| 6000 Ada payback | 7,160 h → 20 mo | $7,590 ÷ $1.06/hr | computed | 26 Oct 2026 |
| A6000 payback | 8,961 h → 25 mo | $5,108 ÷ $0.57/hr | computed | 26 Oct 2026 |
| 4090 at retail median | 18 mo | $3,100 ÷ $0.47/hr | computed | 26 Oct 2026 |

Buy-side prices (3090 $1,206 sold, 4090 $2,214 sold, 5090 $4,843 sold, A6000 $5,108 ask, 6000 Ada $7,590 ask, PRO 6000 $14,400) come from the ownership ledger report: [`2026-10-14-gpu-price-ledger.md`](2026-10-14-gpu-price-ledger.md). Retail medians ($3,100 4090; $10,449 PRO 6000) are GPU Economy's, same read.

## The findings behind the numbers

1. **New silicon clears above its rate cards, old silicon below.** H200 $6.30 transacted vs $4.56 listed median (+38%), B200 $9.70 vs $6.99 (+39%). H100 $3.03 vs $3.54, A100 $1.09 vs $2.00, PRO 6000 $1.53 vs $2.32 — all below. One month of data; the pattern is stated, no mechanism claimed.
2. **One card is not one price.** H100 spans 6.2× across 41 providers. Hyperscaler premium on H100: +201% over specialists (GPU Economy's figure).
3. **$/GB-hour reorders the market.** A6000 1.19¢ → A100 1.36 → PRO 6000 1.59 → 4090 1.96 → 3090 2.08 → 6000 Ada 2.21 → 5090 2.62 → B300 3.12 → L40S 3.17 → H100 3.79 → H200 4.47 → B200 5.39. A 4.5× spread, dearest card not the fastest per gigabyte.
4. **Cheapest card to buy pays back fastest.** 7 months (3090 used) to 25 months (A6000). Rental income is what keeps a six-year-old card's street price off zero.
5. **Buying used shortens payback.** 4090: 13 months at $2,214 sold vs 18 at $3,100 retail median.

## Tiers

| tier | what it is | source |
|---|---|---|
| **R1** transacted | volume-weighted winsorized mean of executed trades | Ornn Data OCPI |
| **R2** listed on-demand | provider rate card | GPU Economy hourly scrape |
| **R3** spot | interruptible / reclaimable | GPU Economy |
| **R4** committed | reserved with a minimum term | GPU Economy |
| **R5** community/shared | third-party hosts, reclaimable — cheapest row, different product | GPU Economy |

R1 > R2 > tails; R5 quarantined from every median. Listed medians are per-provider (each provider once at its cheapest region), matching the source's published figure; a median over raw rows would count a provider once per region. `n` is providers.

**Payback convention:** 50% of a 730-hour month = 365 billable hours. Rental fee only — power, hosting, cooling, resale and idle time excluded, so real payback is longer.

## Corrections log

- Listed medians were first computed as median-of-rows (a provider with N regions counted N times). Replaced with the source's published per-provider median; the row-median is retained as `listed_median_allrows_usd_hr` for the disagreement record.
- Payback divisor was first 365×0.5 = 182.5 h/month, making every payback 2× too long. Corrected to 730×0.5 = 365, which reproduces GPU Economy's own published buy-vs-rent months (5090: ours 15.8 with a sold price vs their 17.9 with a retail median, same formula).

## Gaps

- Ornn history past 3 months is paid. H200/B200/PRO 6000/H100 YTD rests on a16z's print, labelled as a citation in every row.
- No rental tier exists for DGX Spark, Mac Studio or MacBook — not rentable as GPU-hours.
- Second transaction index would settle whether R1's premium over listed rates on new silicon is one network's skew.
