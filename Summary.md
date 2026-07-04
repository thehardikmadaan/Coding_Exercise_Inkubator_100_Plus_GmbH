# Equity Analysis Exercise — Summary & Examiner Prep

## Phase 1: Load & Inspect Data

**What was done:**
- Loaded `prices.csv` (26,100 rows, 100 stocks, daily OHLCV for 2021) via a `load_data()` function with a path resolved relative to the notebook location, not the working directory (robust to however Jupyter is launched).
- Discovered the CSV header was corrupted: the first column name literally contained `\usepackage{ulem}date` instead of `date` — a stray LaTeX command prepended to the column name. This broke `parse_dates=["date"]` since pandas couldn't find a column matching that string.
- **Fix:** loaded the CSV without `parse_dates`, renamed column 0 to `"date"` regardless of its corrupted prefix, then converted to datetime afterward. More robust than trying to string-match or strip the exact garbage text.
- Did an initial visualization (line plot of sample tickers' close prices) as a sanity check on data shape.

**Why it matters:** without this fix, the notebook wouldn't run at all. This is the kind of "silent blocker" an examiner wants to see you diagnose methodically rather than guess-and-check.

---

## Phase 1.5: Data Quality — Outliers

**What was found:**
- `df.describe()` revealed `high` and `close` had implausibly large max values (6632 and 6502) compared to `open`/`low` (max ~1797/1778).
- Investigated via `df.loc[df["high"].idxmax()]` → found a single row: **Microsoft, 2021-11-15**, where `high`/`close` were ~11x larger than `open`/`low` and ~11x larger than the days immediately before/after.
- Confirmed by checking neighboring days — the values divided by 10 fit smoothly back into Microsoft's normal trading range.
- Repeated this same detection process and found the identical ~11x pattern recurring: **Intel (2021-06-07)**, **Coca-Cola (2021-02-19)**, then generalized to a systematic scan across the whole dataset (`high`/`close` > 3× `open`/`low` max) which caught **7 more**: AT&T, Amazon, Apple, Cisco, ExxonMobil, Ford, Pfizer.
- **Total: 10 corrupted rows out of 26,100**, all sharing the exact same signature (~11x inflation, only on `high`/`close`, `open`/`low` unaffected).
- **Fix applied:** divided the affected `high`/`close` values by 10 for all 10 rows, in one blanket operation based on the systematic filter.
- Verified afterward: re-ran the scan, zero rows remained above the 3x threshold; also checked each of the 10 affected tickers individually for any other suspicious rows (none found).

**Important caveat to state explicitly:** the "divide by 10" fix is an *inferred* correction based on pattern-matching (the corrected values fit smoothly with `open`/`low` and neighboring days) — not verified ground truth, since there's no way to know the actual original value with certainty.

**Why it matters:** 5 of these 10 rows (FORD, EXXO, AMAZ, CISC, PFIZ) fell inside the June 1–Oct 13 analysis window. Left uncorrected, they risked producing false top-5 performers based on corrupted single-day prices.

**Still open / not yet done:** 23 missing (`NaN`) values scattered across `open/high/low/close/volume` — identified in Phase 1 but not yet located or addressed. Worth doing before finalizing, in case any land on your window boundaries for the top 5 stocks.

---

## Phase 2: Performance Calculation (Top 5 Movers)

**Approach:**
- Filtered to the window `2021-06-01` to `2021-10-13`.
- Used `groupby("isin").first()` / `.last()` on the close price within that window, rather than assuming every stock trades on exactly June 1 and Oct 13. This also has a useful side effect: `.first()`/`.last()` as pandas aggregations skip `NaN` automatically per column, so a missing price exactly on the boundary date wouldn't silently break the calculation.
- Computed `% return = (end_close - start_close) / start_close * 100`.
- Took the top 5 by `return_pct`.

**Result (confirmed correct):**

| Name | Ticker | ISIN | Return % |
|---|---|---|---|
| General Electric | GENE | XX0000000010 | 940.00% |
| Netflix | NETF | XX0000000028 | 180.00% |
| Adobe | ADOB | XX0000000036 | 141.86% |
| Nokia | NOKI | XX0000000044 | 110.00% |
| NVIDIA | NVID | XX0000000051 | 85.00% |

**Validation steps taken:**
- GE's 940% return looked suspiciously close to the outlier magnitude (~10x) we'd just been fixing — checked both boundary dates (late May and mid-Oct) and confirmed smooth, gradual price movement at both ends. Genuine result, not a data error.
- Checked ranks 6–10 to confirm a clean separation gap after NVIDIA (85.0% vs. next-highest 69.85%) — confirms the top 5 is not sensitive to any small remaining data noise.

**Interesting observation for write-up:** four of the five top returns (940.0, 180.0, 110.0, 85.0%) are suspiciously close to clean round numbers, while Adobe (141.86%) is not. Strong signal that this synthetic dataset has a handful of "designed winner" stocks with hardcoded target returns, while the rest move more organically. Ranks 6–10 show no such round-number pattern, supporting this theory.

---

## Phase 3: 30-Day Average Volume

**Judgment call flagged upfront:** "30 days" is ambiguous — calendar days vs. trading days. Chose **30 trading days** (the 30 most recent rows of actual data on/before Oct 13), since that's the standard meaning of "N-day average volume" in finance (e.g., stock screeners always mean trading days, not calendar days), and it's more stable across stocks/periods since it isn't affected by how many holidays happen to fall in a given calendar window.

**Approach:**
- Queried the **full `df`**, not the `window` variable from Phase 2 — the 30-trading-day lookback from Oct 13 reaches back before June 1, so filtering to the June–Oct window first would have silently truncated the lookback and produced a wrong average with no error.
- Sorted by date before `.tail(30)` — `.tail()` takes the last N rows *as currently ordered*, not chronologically, so sorting first is required to reliably get "the 30 most recent trading days."
- Added a `days_used` self-check column alongside the result. This isn't part of the answer itself — it verifies the assumption that 30 rows of history actually existed for each stock before Oct 13. If any stock showed fewer than 30, that would mean a data gap or short history, worth flagging rather than silently averaging over less data than intended.

**Result (confirmed — all 5 stocks show `days_used == 30`, so no gaps):**

| Name | Ticker | ISIN | Return % | 30-Day Avg Volume |
|---|---|---|---|---|
| General Electric | GENE | XX0000000010 | 940.00% | 8,167,106 |
| Netflix | NETF | XX0000000028 | 180.00% | 4,974,503 |
| Adobe | ADOB | XX0000000036 | 141.86% | 3,108,859 |
| Nokia | NOKI | XX0000000044 | 110.00% | 5,131 |
| NVIDIA | NVID | XX0000000051 | 85.00% | 8,284,994 |

**Note:** Nokia's average volume (~5,131) is orders of magnitude smaller than the others — consistent with what was already observed in Phase 1 (Nokia is a genuinely low-priced, low-volume stock in this dataset, confirmed by inspecting its early trading days). Not a data error, just a naturally different-scale stock.

**General principles reinforced here (useful beyond this exercise):**
- For any ambiguous term (like "30 days"), identify the domain-standard interpretation and state the choice explicitly rather than picking one silently.
- Build self-check columns into calculations that depend on an assumption (e.g., "there will be 30 rows available") — cheap to add, catches silent failures that wouldn't otherwise throw an error.
- Filter data to the *minimum scope actually needed for that specific calculation* — reusing a subset built for a different purpose (like the Phase 2 `window`) can silently break calculations that need a wider range.

---

## Phase 3.5: Missing Values (23 NaNs) — Located & Fixed

**What was found (all 23 accounted for, no other hidden gaps):**
- **Adobe (ADOB):** 1 missing row, exactly on `END_DATE` (2021-10-13). All of `open/high/low/close/volume` NaN.
- **Netflix (NETF):** 22 consecutive missing trading days, covering all of July 2021 (July 1–30). Same columns NaN throughout.

**Initial check before fixing:** confirmed `groupby("isin").last()` (used in Phase 2) skips NaN automatically per column, so Adobe's original `end_close` (1026.4742) had already been silently substituted with **Oct 12's** close instead of a broken Oct 13 value — verified by checking the raw rows around that date. This was a lucky safety net, not something deliberately engineered, and worth stating precisely rather than assuming every stock's "end of window" price was in fact from Oct 13.

**Decision: fix via linear interpolation, per ticker.** Reasoning:
- Netflix's gap is a smooth, continuous time series interrupted by a data gap — not random scattered missingness — so a straight-line estimate between the last known price (June 30) and the next known price (Aug 2) is a defensible inference, unlike inventing values with no basis.
- **Critical implementation detail:** interpolation was done using `groupby("isin").transform(...)`, ensuring each stock's series is interpolated independently — interpolating across ticker boundaries (i.e., on the whole column at once, sorted only by date) would blend one stock's prices into another's gap, which would be wrong.

```python
df = df.sort_values(["isin", "date"]).reset_index(drop=True)
price_cols = ["open", "high", "low", "close", "volume"]
df[price_cols] = df.groupby("isin")[price_cols].transform(lambda s: s.interpolate(method="linear"))
```

**Verification:** checked the interpolated Netflix values (late June–early July) — smooth, monotonic progression with no erratic jumps. Confirmed via `df.info()` afterward that all 26,100 rows now have complete `open/high/low/close/volume` data (zero remaining NaNs).

**Caveat to state explicitly:** like the earlier ~11x outlier fix, interpolated values are an *estimate*, not real observed data — for Netflix in particular, an entire month of "history" is synthetic filler, not something that happened. This should be disclosed, especially since it directly affects the visual price plot (a full month of Netflix's chart is generated, not observed).

**Effect on results — small, expected shift in Adobe only (others unaffected):**

| Name | Ticker | Return % (before fix) | Return % (after fix) |
|---|---|---|---|
| Adobe | ADOB | 141.86% | **141.49%** |

Adobe's `end_close` changed from 1026.4742 (Oct 12 substitute) → 1024.90655 (interpolated Oct 13 value). Ranking order unaffected — Adobe remains comfortably 3rd. Netflix's own return (180.00%) was unaffected since its gap sits in the *middle* of the window, not at a boundary used for the return calculation.

**Final, fully-verified top 5 (after all fixes: outlier correction + interpolation):**

| Name | Ticker | ISIN | Return % | 30-Day Avg Volume |
|---|---|---|---|---|
| General Electric | GENE | XX0000000010 | 940.00% | 8,167,106 |
| Netflix | NETF | XX0000000028 | 180.00% | 4,974,503 |
| Adobe | ADOB | XX0000000036 | **141.49%** | **3,084,781** |
| Nokia | NOKI | XX0000000044 | 110.00% | 5,131 |
| NVIDIA | NVID | XX0000000051 | 85.00% | 8,284,994 |

---

## Still To Do

- **Plot:** price history of the top 5 over the June–October window. Note: Netflix's July segment will now show interpolated (estimated) rather than observed values — worth a visual annotation or at least a written disclosure.
- **PDF deliverable:** table of name, ISIN, performance %, 30-day avg volume for the top 5 (numbers above, now final).
- **Write-up:** consolidate all judgment calls and observations above.

---

## What an Examiner Might Ask

**About your data cleaning process:**
- *"How did you decide something was an error versus a genuine extreme value?"* → Be ready to explain the domain-logic check (high should never be wildly larger than open/low/close on the same row) plus the neighboring-day consistency check, rather than a purely statistical threshold.
- *"Why divide by 10 specifically — how do you know that's the correct original value?"* → Be honest: it's an inference based on the corrected value fitting the surrounding trend, not verified ground truth. This is often exactly what they want to hear — they're testing whether you overstate confidence in a guess.
- *"What if you were wrong about the fix — how would that affect your results?"* → Know which of your top-5/top-10 stocks depended on a corrected value, and whether the ranking would survive if that row had been excluded instead of corrected.
- *"Why not use a simple `df.dropna()` or global outlier removal?"* → Explain why row-level or global exclusion could lose valid data unnecessarily, and why targeted correction based on evidence was more appropriate here.

**About your methodology choices:**
- *"Why `close` price for performance, not `open` or an average?"* → Be ready to justify this (close is the standard end-of-day reference price used for performance calculations in finance).
- *"How did you handle stocks that don't trade exactly on your start/end date?"* → Explain the `groupby().first()/.last()` approach and why it's more robust than assuming exact date matches.
- *"What would you do differently with more time?"* → Good answer: systematic outlier detection from the start (rather than discovering it via `.describe()` one max at a time), and a full audit of missing values before Phase 2.

**About your general data instincts:**
- *"What was the most interesting thing you found in this dataset?"* → The recurring 11x error pattern across 10 unrelated stocks, and the round-number synthetic return pattern in the top performers.
- *"How confident are you in your top 5 — could this change with different assumptions?"* → Reference the clean gap between rank 5 and rank 6 as evidence the ranking is robust, not borderline.

**Practical/technical:**
- *"Walk me through your code without me looking at it."* → Make sure you can narrate each phase (load → clean → filter/compute → rank) fluently, including why each step exists.
- *"What assumptions are baked into your 30-day volume calculation?"* (once you do Phase 3) → E.g., does "30-day" mean 30 calendar days or 30 trading days? This is exactly the kind of ambiguity they expect you to flag, not silently resolve.