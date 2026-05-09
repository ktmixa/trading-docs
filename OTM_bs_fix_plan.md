# OTM Put Overlay: Replacing Black-Scholes with Historical Prices

**Purpose:** Retest the XSP put overlay using real historical bid/ask data instead of BS
pricing, to quantify how much the original research overstated hedge value.

---

## 1. What the Original BS Model Gets Wrong

The current backtest prices XSP puts via Black-Scholes with σ = VIX/100 × 1.3. This
produces two compounding biases, both in the optimistic direction:

**Bias 1 — Skew (entry cost understated)**
BS assumes a flat vol surface. Real SPX puts at 30% OTM consistently trade at 1.5–2× the
ATM implied vol due to persistent left-tail skew. Using VIX × 1.3 as σ still misses this
by 20–50%. The puts cost more than the model thinks.

**Bias 2 — Bid-ask blowout in stress (exit value overstated)**
When V52 signals cash (regime closing), that coincides with rising volatility and falling
liquidity. Bid-ask spreads on deep OTM puts widen sharply in stress — exactly when you
try to EXIT. Selling at mid in a crash is not executable. You sell at bid, which can be
50–75% of mid during a fast market.

These two biases act in opposite directions for cost estimation but the same direction for
"does this hedge work as advertised": you pay more to enter than modeled, and receive less
on exit than modeled.

---

## 2. Data Sources

### 2a. XSP Monthly Option Chains (primary pricing source)

**Location:** `/home/brewadmin/projects/trading/data/options_data/XSP_*_eod.parquet`

**Coverage:** 39 monthly expiries, January 2023 → April 2026.

**Structure:** One file per expiry date. Each file contains daily EOD snapshots
(`created` column) from approximately 88–92 days before expiry through expiry day.
Columns include `strike`, `right`, `bid`, `ask`, `volume`, `count` per snapshot.

**Snapshot depth advantage:** XSP files capture 88–92 DTE on first snapshot, covering the
90-DTE entry window directly. SPX files only reach ~60 DTE on first snapshot — XSP chains
are used preferentially because they eliminate any DTE extrapolation at entry.

**Strike range:** XSP $5 increments at 30% OTM. For example, at XSP 560, target strike is
560 × 0.70 = 392 → rounded to 390.

**Market regimes covered:**
- 2023: post-rate-hike rally, VIX declining from 20→13
- 2024: rate-cut cycle, low-vol bull
- 2025: tariff shock (April 2025), VIX spike to 52
- 2026: partial recovery

**SPX chains** (`SPX_*_eod.parquet`, 42 expiries) are available as a cross-check and
fallback if a specific XSP strike/date is missing. SPX prices divide by 10 to get XSP
equivalent (same index, different scale).

### 2b. SPY Weekly Option Chains — COVID Stress Period

**Location:** `/home/brewadmin/projects/trading/data/options_data/SPY_*_eod.parquet`

**Coverage:** 49 daily files covering 2020-01-03 → 2020-03-31 (weekly expiries), plus
37 monthly expiries from 2023-01 through 2026-03.

**Purpose:** SPY ≈ SPX/10 (ETF tracking S&P 500). SPY option prices at 30% OTM are
directly comparable to XSP prices. The 2020 Jan–Mar files capture:
- Pre-crash baseline (Jan 2020): tight spreads, normal liquidity
- Fast crash onset (Feb 24 – Mar 6, 2020): spreads beginning to widen
- Peak stress (Mar 9–20, 2020): VIX 60–85, maximum bid-ask blowout, near-zero bids on
  deep OTM puts

This is the ground truth for stress execution friction. It will be used to:
1. Characterize spread blowout as a function of VIX level
2. Apply those stress multiples to the SPX 2025 tariff-shock scenarios
   (SPX chain data exists but SPY 2020 gives the more extreme stress benchmark)

### 2c. ES Futures 1-Min Bars (underlying price reference)

**Location:** `/home/brewadmin/projects/trading/data/futures_bars/ES/1min_db/`

**Coverage:** 2018-12-31 → 2026-05-01.

**Purpose:** Daily close of ES / 10 ≈ XSP underlying price. Used to:
- Verify the strike selected by the formula at the time of entry
- Cross-check that the SPX chain file's snapshot date aligns with the intended observation

---

## 3. Coverage Scope and Pre-2023 Fallback

**Historical pricing applies to 2023–2026 only.** XSP monthly chain files begin
January 2023. The V52.DD12 backtest starts 2000 (synthetic SSO pre-inception) and runs to
present. Any ENTER/ROLL/EXIT events before January 2023 have no observable XSP bid/ask and
remain on Black-Scholes pricing (σ = VIX/100 × 1.3). These are clearly labeled "BS proxy"
in all output tables.

**Snapshot depth:** XSP chain files capture 88–92 DTE snapshots per monthly expiry —
sufficient to price a 90-DTE put directly at entry without extrapolation. The 60-DTE
data-gap concern that would apply to SPX chains does not arise here. All ENTER prices are
labeled "90-DTE actual" when an XSP snapshot within ±5 calendar days of 90 DTE is
available; otherwise "nearest snapshot (N DTE)" with N stated.

**Key coverage facts:**
- XSP monthly chains: Jan 2023 → Apr 2026 (39 expiries, 88–92 DTE depth)
- SPY weekly chains: Jan–Mar 2020 (49 files, COVID stress calibration)
- ES 1-min bars: Dec 2018 → May 2026 (underlying price reference)
- Pre-2023 V52.DD12 signal events: BS pricing only (labeled)

---

## 4. Simulation Architecture

### 4a. Event Types

The simulation is driven by the V52.DD12 signal from the existing backtest output
(`backtest/run_walk_forward.py` results or a fresh signal run). Four events:

| Event | Trigger | Action |
|---|---|---|
| ENTER | V52 transitions cash → UPRO | Buy 90-DTE XSP puts at ask |
| ROLL | 60 calendar days since last purchase | Sell old at bid, buy new 90-DTE at ask |
| EXIT | V52 transitions UPRO → cash | Sell all puts at bid (urgent) |
| HOLD | No transition, no roll due | Mark-to-market at mid, no trade |

### 4b. Strike Selection

On any ENTER or ROLL-buy day:

```python
xsp_close = es_close / 10      # ES close ÷ 10 ≈ XSP underlying
target_xsp_strike = floor(xsp_close * 0.70 / 5.0) * 5.0   # XSP uses $5 increments at 30% OTM
```

Then look up the matching row in the XSP chain file for the target expiry
(files at `/data/options_data/XSP_{expiry}_eod.parquet`):
`(expiration == target_expiry) & (strike == target_xsp_strike) & (right == 'PUT') &
(created.date() == observation_date)`.

Select the snapshot with DTE closest to 90 (should be 88–92 DTE from the first available
snapshot in the chain file). Record actual DTE in the output.

### 4c. Entry Execution Price (ENTER / ROLL-buy)

```python
entry_px = xsp_ask             # XSP price directly — no conversion needed
entry_px += 0.05               # current runner's +$0.05 limit buffer, kept as-is
```

For pre-2023 events where no XSP chain is available, fall back to BS pricing and label
the event "BS proxy" in output.

### 4d. Exit Execution Price (EXIT / ROLL-sell)

```python
exit_px = xsp_bid              # XSP bid directly — sell at bid, not mid
```

In stress conditions (VIX > 30 at time of exit), apply a haircut to the observable bid
using the SPY 2020 stress calibration (see Section 5).

### 4e. Contract Sizing

```python
nav = 100_000           # paper account starting NAV (or actual IBKR NAV if paper)
budget = ANNUAL_BUDGET * (ROLL_INTERVAL / 365.0) * nav   # = $164 per 60-day period
contracts = max(1, int(budget / (entry_px * 100)))
```

This matches the live runner exactly.

### 4f. Daily Mark-to-Market

On each HOLD day, look up the mid price for the current position:

```python
mid = (xsp_bid + xsp_ask) / 2         # XSP mid directly
position_value = contracts * mid * 100
```

Track cumulative premium spent and current mark value. Report net P&L of the overlay
separately from the equity leg.

---

## 5. Stress Execution Model (Bid-Ask Blowout)

### 5a. Calibrate from SPY 2020 Data

For each SPY 2020 daily snapshot, compute:

```python
mid = (bid + ask) / 2
spread_pct = (ask - bid) / mid    # as fraction of mid
```

For puts at 25–35% OTM in the 2020 data, extract the spread_pct series day by day and
align it to VIX level on that date (VIX is available from yfinance or the VIX daily data).

Expected result is a lookup table:

| VIX range | Typical spread % of mid |
|---|---|
| 10–20 | 5–10% |
| 20–30 | 10–20% |
| 30–45 | 25–40% |
| 45–60 | 40–60% |
| 60+ | 60–80% (near-zero bids) |

### 5b. Apply Haircut on Stressed Exits

When an EXIT fires and VIX > 30 (stress threshold), apply the calibrated haircut to the
observable bid:

```python
stress_factor = lookup_spread_pct(vix_at_exit)
adjusted_bid = observable_bid * (1 - stress_factor / 2)  # conservative: half the spread
exit_px = max(adjusted_bid, 0.01)    # floor at $0.01 XSP ($1 SPX)
```

Use the `adjusted_bid` for exits and for the ROLL-sell side during stress periods.
Report the "no-haircut" case alongside as a bound.

### 5c. Near-Worthless Skip Logic Validation

The live runner skips the sell leg of a ROLL if `bid < SKIP_SELL_THRESHOLD ($0.02 XSP)`.
Validate this from data: on each ROLL-sell day, check whether the observable bid is above
or below $0.02 XSP ($0.20 SPX). Count how often the skip fires and what value is abandoned
when it does. Confirm the threshold is set correctly.

---

## 6. Comparison Baseline

Run BS pricing alongside the historical-price simulation using the same V52 signal dates:

```python
bs_price = black_scholes_put(S=xsp_price, K=strike, T=dte/365, r=irx/100,
                              sigma=vix/100 * 1.3)
```

For each ENTER/EXIT/ROLL event, record both the BS price and the historical ask/bid. The
difference is the "skew gap" and "execution gap" respectively.

Report three scenarios side by side:
1. **BS model** (original research baseline)
2. **Historical mid** (replace BS with mid; removes skew bias, keeps execution optimism)
3. **Historical bid/ask with stress haircut** (full fix — buys at ask, sells at bid with stress adjustment)

---

## 7. Output Metrics

### 7a. Per-Event Table (2023–2026 historical window)

For each ENTER/ROLL/EXIT event in the 2023–2026 window:
- **Skew gap**: BS entry price vs actual ask, by VIX level
- **Stress execution gap**: mid vs bid on exit, by VIX level
- **SKIP_SELL fires**: count and $ abandoned

### 7b. Named Case Study — COVID 2020 Exit

The COVID crash exit (signal date ≈ 2020-02-27, SPY $271.88, estimated put strike ~190) is
the canonical stress test of the overlay. This event lies outside the 2023–2026 historical
window, so it stays on BS pricing for the entry cost — but SPY 2020 bid/ask data is
available for the EXIT side (late February / early March 2020 weekly chains). Produce:

- **SPY Feb 27 2020 chain lookup**: actual bid on the ~190 strike put for the nearest expiry
- **Spread at exit**: observable SPY bid vs mid vs BS estimate on that date
- **VIX at exit**: ~40 → apply stress calibration from Section 5
- **Verdict**: did the overlay pay out (put value > 2× cumulative premium paid to that date)?

Label this "COVID 2020 Exit Case Study" in `scenario_comparison.md`. It demonstrates the
value of the stress execution model even where entry price must use BS.

### 7c. Aggregate Overlay P&L and Equity Curve Integration

Across the full backtest (2000–2026, pre-2023 on BS, 2023–2026 on historical):

- **Annual overlay cost** ($ and % of NAV) — broken out by year
- **Total premium spent** vs **total monetization proceeds**
- **Net drag** on strategy CAGR
- **Payout events**: which V52 exits coincided with put value > 2× cost

Then integrate the overlay P&L into the V52.DD12 equity curve:

```python
# Each day: adjusted_nav = equity_nav + put_position_value - cumulative_premium_paid
# On EXIT days: adjusted_nav reflects both equity gain/loss and put monetization
adjusted_cagr, adjusted_maxdd = compute_stats(adjusted_equity_curve)
```

Report side by side:
| Metric | V52.DD12 (no overlay) | + Overlay (BS) | + Overlay (historical bid/ask) |
|---|---|---|---|
| CAGR | 19.1% | ? | ? |
| MaxDD | 51.5% | ? | ? |
| Calmar | ? | ? | ? |

This is the primary output: does the overlay improve Calmar (CAGR/MaxDD) after realistic costs?

---

## 8. Files to Produce

| File | Description |
|---|---|
| `backtest/run_otm_bs_fix.py` | Main simulation script |
| `backtest/results/otm_bs_fix/summary.csv` | Per-event price comparison (BS vs actual) |
| `backtest/results/otm_bs_fix/stress_calibration.csv` | SPY 2020 spread_pct vs VIX table |
| `backtest/results/otm_bs_fix/scenario_comparison.md` | Three-scenario summary table |

---

## 9. Limitations and Caveats

1. **Pre-2023 events on BS**: Any ENTER/ROLL/EXIT before January 2023 uses BS pricing
   (σ = VIX/100 × 1.3). The number of such events depends on V52.DD12 signal frequency
   during 2000–2022. They are labeled but not corrected for skew or execution friction.
   The COVID 2020 exit (Section 7b) is the most consequential pre-2023 event and receives
   special treatment via SPY 2020 chain data on the exit side.

2. **SPX vs XSP liquidity**: SPX options are more liquid than XSP. XSP bid-ask spreads
   may be slightly wider than SPX/10, particularly in stress. The SPY 2020 stress data
   (a smaller, more retail instrument) may give a more accurate liquidity proxy for XSP
   than SPX.

3. **EOD snapshots only**: All bid/ask data is captured at EOD (~4:15–4:30 PM ET). The
   morning runner executes at 9:40 AM ET. Opening bid-ask spreads can be wider than EOD,
   especially on high-vol days. This is a residual optimism bias in the execution model.

4. **No IV surface fitting**: The stress haircut is a simple VIX-bucketed multiplier, not
   a full vol-surface model. It captures the magnitude of blowout but not the precise
   smile shape. This is adequate for sizing the bias correction; it is not a production
   pricing engine.
