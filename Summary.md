# Equity Analysis Exercise — Approach & Data Quality Notes

This document tracks the methodology, judgment calls, and data quality issues found while completing the equity analysis exercise. Full code is in `hardik_src/solution.ipynb`.

## Phase 1: Load & Inspect Data

- Loaded `prices.csv` (26,100 rows, 100 stocks, daily OHLCV for 2021) via `load_data()`, with the path resolved relative to the notebook location rather than the working directory.
- **Data issue found:** the CSV header was corrupted — the first column name literally contained `\usepackage{ulem}date` instead of `date` (a stray LaTeX command prepended to the column name). This broke `parse_dates=["date"]`.
- **Fix:** loaded without `parse_dates`, renamed column 0 to `"date"` unconditionally, then converted to datetime. Robust regardless of the exact corrupted prefix.

---

## Phase 2: Data Quality — Outliers

- `df.describe()` showed `high`/`close` with implausibly large max values (6632/6502) vs. `open`/`low` (~1797/1778).
- Traced to **Microsoft, 2021-11-15**: `high`/`close` were ~11x larger than `open`/`low` and ~11x larger than neighboring days. Confirmed by checking days before/after — the values divided by 10 fit smoothly back into the normal range.
- Same ~11x pattern recurred on **Intel (2021-06-07)** and **Coca-Cola (2021-02-19)**, then a systematic scan (`high`/`close` > 3× `open`/`low` max) caught **7 more**: AT&T, Amazon, Apple, Cisco, ExxonMobil, Ford, Pfizer.
- **Total: 10 corrupted rows out of 26,100**, all sharing the identical signature — only `high`/`close` affected, `open`/`low` untouched.
- **Fix:** divided the affected `high`/`close` values by 10 for all 10 rows via one blanket filter. Re-scanned afterward — zero rows remained above the threshold.
- **Caveat:** the divide-by-10 fix is an *inferred* correction based on pattern-matching (corrected values fit smoothly with neighboring days) — not verified ground truth.
- **Why it mattered:** 5 of the 10 rows (Ford, ExxonMobil, Amazon, Cisco, Pfizer) fell inside the June 1–Oct 13 analysis window; left uncorrected, they risked producing false top-5 performers.

---

## Phase 3: Performance Calculation (Top 5 Movers)

- Filtered to the window `2021-06-01`–`2021-10-13`.
- Used `groupby("isin").first()`/`.last()` on close price within the window, rather than assuming every stock trades on the exact boundary dates. This also has a useful side effect: `.first()`/`.last()` skip `NaN` per column automatically, so a missing boundary price doesn't silently break the calculation.
- `% return = (end_close - start_close) / start_close * 100`.
- Confirmed the ranking is robust: clean separation gap between rank 5 and rank 6 (no small-margin sensitivity).

---

## Phase 4: 30-Day Average Trading Volume

- **Judgment call:** "30 days" is ambiguous (calendar vs. trading days). Used **30 trading days** (the 30 most recent rows of data on/before Oct 13) — the standard convention for "N-day average volume" in financial tools.
- Queried the **full dataset**, not the Phase 3 window — the 30-trading-day lookback from Oct 13 extends before June 1, so filtering to the analysis window first would have silently truncated it.
- Sorted by date before `.tail(30)` — `.tail()` takes the last N rows as currently ordered, not chronologically.
- Added a `days_used` self-check column to confirm 30 rows of history actually existed for each stock (all 5 top performers returned `days_used == 30`).

---

## Phase 5: Missing Values (23 NaNs) — Located & Fixed

- **Found:** Adobe (ADOB) missing exactly 1 row, on 2021-10-13 (the analysis end date). Netflix (NETF) missing 22 consecutive trading days, covering all of July 2021.
- Verified `groupby().last()` (used in Phase 3) had already silently substituted Adobe's Oct-12 close for the missing Oct-13 value.
- **Fix:** linear interpolation, grouped independently per ticker (`groupby("isin").transform(...)`) — critical to avoid blending one stock's prices into another's gap.
- **Caveat:** interpolated values are estimates, not observed data — for Netflix, a full month of "history" is synthetic filler. Disclosed visually in the plot (dashed segment) and here.
- **Effect on results:** Adobe's return shifted slightly (141.86% → 141.49%, using the interpolated Oct-13 value instead of the Oct-12 substitute). Ranking order unaffected.

---

## Phase 6: General Electric — Investigated, Treated as a Reverse Stock Split

- GE showed a 940% return, driven by `open/high/low/close` all shifting together by ~8x on 2021-08-02, holding steady afterward (ruled out as a repeat of the Phase 2 outlier bug — that pattern was isolated to `high`/`close` only, on single rows, reverting the next day).
- **Volume check:** volume dropped sharply on the same date — from ~22M–69M/day before Aug 2 to ~5M–12M/day after. Simultaneous price-up/volume-down on the same date is the signature of a reverse stock split (fewer shares outstanding after the split).
- **Decision:** treated as a 1-for-8 reverse split and adjusted — multiplied pre-split prices by 8, divided pre-split volume by 8 — so the historical baseline reflects true organic performance rather than a mechanical split effect.
- **Caveat:** this is an inferred interpretation, not a certainty. The volume ratio isn't a perfectly clean 8x (observed ~4–6x), and there's no external record to confirm against since the dataset is synthetic. An equally coherent alternative — a deliberately designed "jump" mechanism for a synthetic winner stock — was considered and not fully disproven, only outweighed by the volume evidence.
- **Effect:** after adjustment, GE's return drops below 30%, removing it from the top 5. Lockheed Martin (LOCK) takes the 5th spot.

---

## Final Results

| Name | Ticker | ISIN | Return % | 30-Day Avg Volume |
|---|---|---|---|---|
| Netflix | NETF | XX0000000028 | 180.00% | 4,974,503 |
| Adobe | ADOB | XX0000000036 | 141.49% | 3,084,781 |
| Nokia | NOKI | XX0000000044 | 110.00% | 5,131 |
| NVIDIA | NVID | XX0000000051 | 85.00% | 8,284,994 |
| Lockheed Martin | LOCK | XX0000000531 | 69.85% | 17,737,050 |

Nokia's volume is genuinely orders of magnitude smaller than its peers — confirmed not to be a data error, just a naturally low-priced, low-volume stock in this dataset.

---

## Additional Observations

- **Signs of synthetic data design:** several top performers show suspiciously round-number returns (180.00%, 110.00%, 85.00%; GE's pre-adjustment 940.00% also fit this pattern). This strongly suggests the dataset was built with hardcoded target returns for a subset of "winner" stocks, implemented via discrete price jumps rather than smooth organic movement. Outside these designed rows, the remaining data shows realistic, non-round day-to-day volatility.
- **The 10 corrupted outlier rows** all shared an identical ~11x multiplier across unrelated stocks — more consistent with a systematic generation bug than independent real-world data errors.