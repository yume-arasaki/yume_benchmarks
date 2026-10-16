# GPU price ledger — US market, 2026

Receipts for the price post (`2026-10-14`, "is this a shortage or a Labubu moment"). Every number printed in the post maps to a row in this ledger. Tiers and rules are at the bottom.

Raw data: [`data/gpu-prices/observations.csv`](../data/gpu-prices/observations.csv) · [`ledger.jsonl`](../data/gpu-prices/ledger.jsonl) · aggregated [`series-monthly.csv`](../data/gpu-prices/series-monthly.csv)

Chart: [`assets/gpu-price-matrix-2026-10.png`](../assets/gpu-price-matrix-2026-10.png)

## Headline numbers, and where each one comes from

| Number in the post | Value | Source | Tier | Window |
|---|---|---|---|---|
| RTX 3090 sold | $1,206, +56% | GetPCParts weekly sold series | T4 sold | 17 Jan → 6 Oct 2026 |
| RTX 4090 sold | $2,214, −6%, still +38% over launch | GetPCParts weekly sold series | T4 sold | 17 Jan → 6 Oct 2026 |
| RTX 5090 sold | $4,843, +70% | GetPCParts weekly sold series | T4 sold | 17 Jan → 6 Oct 2026 |
| RTX A6000 ask | $5,108, +28% since Jul | Silicon Comps daily asking median | T3 ask | 28 Jul → 11 Oct 2026 |
| RTX 6000 Ada ask | $7,590, +28% since Jul | Silicon Comps daily asking median | T3 ask | 28 Jul → 11 Oct 2026 |
| RTX PRO 6000 | $14,400 new | Micro Center in-store listing (SKU 44594) | T2 street | 14 Oct 2026 |
| RTX PRO 6000 launch listings | ~$8,565 | GPU Poet, from Connection launch listings, corroborated by TweakTown and Tom's Hardware | T1 est. | Mar 2025 |
| DGX Spark list | $6,950 | The Register + NVIDIA marketplace | T1 list | 2 Oct 2026 |
| DGX Spark previous list | $4,699 | NVIDIA forum announcement | T1 list | 23 Feb 2026 |
| DGX Spark launch list | $3,999 | NVIDIA launch announcement | T1 list | 15 Oct 2025 |
| DGX Spark used ask | $8,795 median, n=18 | vramglass eBay tracker | T3 ask | 14 Oct 2026 |
| DGX Spark reseller new | $9,899 | Newegg marketplace listing N82E16883986001, seller Halfmoon Tech | T3 ask, n=1 | 14 Oct 2026 |
| M3 Ultra list raise | $3,999 → $5,299 | World Price Index archive | T1 list | 26 Jun 2026 |
| M3 Ultra 512GB option deleted | — | MacDailyNews, Overcentral | press | Mar 2026 |
| Mac Studio M5 Ultra 256GB | $9,499 | Apple configurator, read by openclawdc | T1 list | 21 Sep 2026 |
| Mac Studio M5 Ultra 96GB | $6,799 (80-core) | Apple configurator, read by Context Studios | T1 list | 24 Sep 2026 |
| MacBook Pro 16 M5 Max 128GB | $5,099 | Apple configurator, read by openclawdc | T1 list | 21 Sep 2026 |
| M3 Ultra used ask | $7,456 median, n=120 | Hard Bargains | T3 ask | 3 Oct 2026 |
| 5090 ask vs sold gap | $6,786 vs $4,843 = $1,943 | Silicon Comps vs GetPCParts | T3 vs T4 | Oct 2026 |
| NVLink spec | 112.5 GB/s, last GeForce card | NVIDIA GPU comparison table, cross-checked against fibermall, rtxinference, Level1Techs | spec | current |
| PCIe 4.0 x16 | ~31 GB/s per direction | same | spec | current |
| Two Sparks for 405B | ConnectX pairing, $13,900 | Newegg listing copy (405B) + 2 × $6,950 | T1 ×2 | Oct 2026 |

## The series as data

Monthly means, with observation counts. Ask series run Jul→Oct because that is where the tracker archive begins.

| RTX 3090 sold | 2026-01 | 2026-02 | 2026-03 | 2026-04 | 2026-05 | 2026-06 | 2026-08 | 2026-09 | 2026-10 |
|---|---|---|---|---|---|---|---|---|---|
| USD | 861 | 927 | 1,030 | 1,084 | 1,063 | 1,125 | 984 | 1,032 | 1,206 |
| n | 3 | 4 | 4 | 3 | 4 | 1 | 5 | 4 | 1 |

| RTX 4090 sold | 2026-01 | 2026-02 | 2026-03 | 2026-04 | 2026-05 | 2026-06 | 2026-07 | 2026-08 | 2026-09 | 2026-10 |
|---|---|---|---|---|---|---|---|---|---|---|
| USD | 2,331 | 2,308 | 2,150 | 2,133 | 2,248 | 2,311 | 2,269 | 2,333 | 2,622 | 2,214 |
| n | 3 | 4 | 4 | 3 | 4 | 4 | 4 | 5 | 4 | 1 |

| RTX 5090 sold | 2026-01 | 2026-02 | 2026-03 | 2026-04 | 2026-05 | 2026-06 | 2026-07 | 2026-08 | 2026-09 | 2026-10 |
|---|---|---|---|---|---|---|---|---|---|---|
| USD | 2,853 | 3,108 | 3,000 | 2,967 | 3,253 | 3,616 | 3,684 | 3,910 | 4,499 | 4,843 |
| n | 3 | 4 | 4 | 3 | 4 | 4 | 4 | 5 | 4 | 1 |

| RTX 3090 ask | 2026-07 | 2026-08 | 2026-09 | 2026-10 |
|---|---|---|---|---|
| USD | 1,345 | 1,524 | 1,818 | 1,818 |
| n | 4 | 31 | 30 | 11 |

| RTX 4090 ask | 2026-07 | 2026-08 | 2026-09 | 2026-10 |
|---|---|---|---|---|
| USD | 2,685 | 2,903 | 3,283 | 3,545 |
| n | 4 | 31 | 30 | 11 |

| RTX 5090 ask | 2026-07 | 2026-08 | 2026-09 | 2026-10 |
|---|---|---|---|---|
| USD | 4,669 | 5,151 | 6,115 | 6,786 |
| n | 4 | 31 | 30 | 11 |

| RTX A6000 ask | 2026-07 | 2026-08 | 2026-09 | 2026-10 |
|---|---|---|---|---|
| USD | 4,000 | 4,289 | 4,960 | 5,108 |
| n | 4 | 31 | 30 | 11 |

| RTX 6000 Ada ask | 2026-07 | 2026-08 | 2026-09 | 2026-10 |
|---|---|---|---|---|
| USD | 5,935 | 5,989 | 7,064 | 7,590 |
| n | 4 | 31 | 18 | 10 |

| DGX Spark ask | 2026-10 |
|---|---|
| USD | 8,820 |
| n | 2 |

## What the two tiers say

Sold (T4) is what buyers actually paid. Ask (T3) is what sellers want. The distance between them is the finding, not noise:

| Card | Ask, Oct | Sold, Oct | Ask premium |
|---|---|---|---|
| RTX 5090 | $6,786 | $4,843 | +40% |
| RTX 4090 | $3,545 | $2,214 | +60% |
| RTX 3090 | $1,818 | $1,206 | +51% |

## Gaps, stated

- **DGX Spark second-hand sold data does not exist** in any systematic public form. eBay does not publish sold prices. What exists is 18 asking listings on one tracker, plus single marketplace listings. The post prints no percentage for it.
- **No systematic Apple second-hand tracker** beyond Hard Bargains, which is a single site.
- **RTX PRO 6000 has no tracker series** — one store price, one street price, one used price, one launch-listing estimate.
- **The NVLink throughput figure** (715 vs 483 tok/s, 2× 3090, vLLM, 7B) is a single published benchmark. The post quotes the spec, not this number, as law.
- **One error row is retained in the ledger** rather than deleted: a $14.55 parts listing on the 3090 sold page, collected 2026-10-14, which the collector now filters by a $300 floor.

## Rules this ledger follows

1. **Every observation is a tuple**: hardware, market tier, metric, value, currency, observation date, source. Prices are perishable and the timestamp is part of the number. The US market moved 15 to 19% per month through 2026.
2. **Four tiers, never mixed**: T1 first-party list, T2 retail street new, T3 second-hand asking, T4 second-hand sold.
3. **Metric hierarchy**: sold median > asking median > lowest listing. The ask-sold gap is recorded as a signal.
4. **Windows differ per source and are labelled**, never averaged together into a single trend.
5. **Gaps are recorded, not filled.**
6. **Single-listing observations are marked n=1** with seller and configuration, so an ask can never pass as a median.

## The collector

The collector, the methodology write-up, and the checks live with the working data (not published here): a weekly sold-listing series, a daily asking series, per-source parsers, dedup on `(source, hardware, metric, date)`, a zero-parse guard that fails loudly rather than recording silence as flat prices, and a price-floor guard against parts listings. Backfilling rehydrates history from each tracker's own archive, so the series can be rebuilt from scratch in one run.
