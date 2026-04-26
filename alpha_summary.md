# MixA Alpha Summary
*Research summary for signal design, backtesting conclusions, and open questions.*
*Period covered: 2000-01-01 → 2026-04-23 (26.4 years). $100k starting capital.*

---

## Current champion: V39 — numbers to beat

**V39 = SMA100/300 + momentum (top 25%) + SPY asymmetric gate + VIX sizing + T-bill yield + adaptive ^IRX reopen + PIT S&P 500 universe + trailing dvol top-50 screen + 10-day trend persistence + SPY recovery re-entry (sma=20, mult=0.25)**

No foreknowledge. All filters real-time computable. Any candidate strategy or universe improvement must clear these numbers to be considered an upgrade.

| Metric | **V39** | H39 (hindsight ceiling) | SPY B&H | QQQ B&H | AOA† |
|--------|---------|------------------------|---------|---------|------|
| End value ($100k start) | **$1,306,185** | $1,228,219 | $775,385 | $815,014 | $555,686† |
| CAGR | **+10.26%** | +10.00% | +8.10% | +8.30% | +6.74%† |
| Sharpe | 0.76 | **0.84** | 0.53 | 0.46 | 0.77† |
| MaxDD | 20.5% | **18.3%** | 55.2% | 83.0% | 28.4%† |
| GFC drawdown | 20.2% | **18.2%** | 55% | 78% | — |
| COVID drawdown | **14.5%** | 17.0% | 34% | 29% | — |
| Rates drawdown | 13.3% | **12.2%** | 24% | 35% | — |
| Win rate | **37.1%** | 36.1% | — | — | — |

*IRA/tax-free, next_bar MOO fill (f=0/0, opening auction price), run 2026-04-25 (data through 2026-04-23). H39 = identical V39 signal on a static hand-picked SP50 — the hindsight ceiling for universe quality.*

†AOA (iShares Core Aggressive Allocation ETF, 80/20 stocks/bonds) from inception 2008-11-11; all others from 2000-01-01.

**How V39 works:**
- *Universe:* Dynamic — historical S&P 500 point-in-time membership (fja05680/sp500 dataset). Only enters tickers that were actually in the index on the entry date; forced-exit on removal. Backtest uses full universe; live trading additionally excludes `WASH_SALE_EXCLUDE` tickers held in juicer.
- *Quality screen:* Top-50 by trailing 30-day dollar-volume; rotates out distressed names as liquidity drains months before bankruptcy
- *Signal:* SMA100/300 trend crossover with 3×ATR stop; 20-day re-entry cooldown
- *Persistence gate:* Requires 10 consecutive LONG signal days before opening — eliminates marginal crossover entries that reverse immediately
- *Momentum filter:* Top 25% by 12-1 month cross-sectional rank
- *Regime filter:* SPY asymmetric gate (close > SMA200 to close, fast or slow reopen gated by ^IRX momentum); VIX halved above 25, paused above 40
- *Recovery re-entry:* When SPY < SMA200 but SPY > SMA20, allows entries at 25% normal size; captures V-shaped recoveries (GFC 2009, COVID 2020, rates 2022) before the full regime gate reopens
- *Cash yield:* Idle cash earns ^IRX (T-bill) rate daily

**T-bill effect:** +0.4pp mean CAGR; +2.1pp in high-rate eras. Models real brokerage/IRA sweep.

**V39 vs H39 (hindsight universe ceiling):** H39 runs the identical V39 signal on a static hand-picked SP50 list — the apples-to-apples hindsight benchmark. Result with corrected MOO fill model: V39 CAGR +10.26% now beats H39 +10.00% — V39 leads on end value and CAGR while H39 leads on Sharpe (0.84 vs 0.76), MaxDD (18.3% vs 20.5%), and stress DDs. The PIT universe's larger pool finds more cyclical winners (energy, financials, industrials across eras) that the static SP50 misses, offsetting the quality penalty. H39's edge is now purely in risk metrics, not raw return — the gap to close is drawdown, not alpha.

**Universe quality and the persistence gate:** V31 (same SP50, no persistence) hit +10.26% CAGR vs H39's +8.10% — ~2.2pp lost to the persistence gate on a clean universe. This is expected: the 10-day persistence filter was designed to filter out marginal names in a noisy PIT universe; on a pre-screened quality list those names don't exist, so persistence mostly delays good entries. **TODO:** once a forward-looking quality universe (third candidate) is built, re-tune or relax `trend_persist_days` — it may be redundant if universe construction already selects for clean-trending names.

**Status:** EOD signal runner scheduled (cron 4:20 PM ET, writes reports to `docs/reports/eod/`). Morning execution runner ready; wires to IBKR once IB Gateway is set up.

---

## Portfolio architecture (V39 — current)

Single-strategy deployment on $100k capital:

| Allocation | Role |
|-----------|------|
| Up to ~83% equity | Long individual stocks in confirmed uptrends (ATR-sized) |
| Remaining cash | Earns 13-week T-bill rate (^IRX) daily |

Universe: dynamic top-50 by trailing 30-day dollar-volume among current S&P 500 members (point-in-time). Forced-exit on index removal. 10-day persistence gate before entry. Recovery mode: 25%-sized entries when SPY < SMA200 but > SMA20. Wash-sale exclusions (`WASH_SALE_EXCLUDE` in `config.py`) applied only in live signal generation.

**MR and UVXY not active.** Mean reversion earned only +0.90% CAGR on its capital slice. UVXY was tested (V37a/b) and was net negative. Both remain documented below.

---

## Strategy 1: Trend-following

### Signal logic

**Entry:** LONG when `close > SMA_fast AND SMA_fast > SMA_slow` (golden cross, daily bars).

**Exit:** FLAT when either condition breaks (signal flip), or when price hits stop-loss.

**Stop-loss:** Static, set at entry: `stop = entry_price − (N × ATR14)`. Never trailed.

**Position sizing:** Risk-normalized. Each trade risks exactly 1% of portfolio value:
```
shares = (portfolio_value × 0.01) / (N × ATR14)
```
High-volatility stocks get fewer shares. Hard cap: no single position > 15% of portfolio.

**Momentum filter:** Cross-sectional 12-1 month momentum rank. Only top 25% eligible for new entries. Existing positions unaffected.

**Regime filters:**
- **SPY regime:** Halt new longs when SPY < SMA200. Asymmetric reopen: fast (price > SMA100) when ^IRX easing, slow (SMA100 > SMA200 golden cross) when tightening.
- **VIX adaptive sizing:** VIX > 25 → half-size. VIX > 40 → skip entirely.

---

## Trend backtest: all variants

V1–V33 used a static curated SP50 universe (hindsight-biased). V34–V39 use the point-in-time S&P 500 universe. H39 is the apples-to-apples hindsight benchmark — V39 signal on the same static SP50. Code retains V34/V35/V36/V39/H39; V1–V33 are in git history.

### V1–V33: static SP50 universe (hindsight reference)

| # | Variant | End value | CAGR | Sharpe | MaxDD | Calmar |
|---|---------|-----------|------|--------|-------|--------|
| 1 | Baseline (SMA50/200, 2×ATR) | $337,691 | +4.7% | 0.38 | 55.1% | 0.08 |
| 4 | + momentum filter (top 25%) | $367,378 | +5.1% | 0.44 | 31.0% | 0.16 |
| 6 | Longer SMA (100/300) + momentum | $886,460 | +8.7% | 0.64 | 40.2% | 0.22 |
| 15 | V4 + SPY regime + VIX sizing | $529,975 | +6.6% | 0.58 | 26.8% | 0.24 |
| 16 | Super (SMA100/300 + SPY gate + VIX sizing) | $1,172,929 | +9.8% | 0.76 | 27.8% | 0.35 |
| 29 | V19 + T-bill yield on idle cash | $1,232,612 | +10.03% | 0.78 | 22.2% | 0.45 |
| **31** | **V29 + adaptive ^IRX reopen** | **$1,303,844** | **+10.26%** | **0.80** | 23.0% | 0.45 |
| 33 | V29 + adaptive TLT reopen (conservative alt) | $1,268,464 | +10.15% | 0.79 | **20.8%** | **0.49** |
| — | SPY buy-and-hold | $750,923 | +7.97% | 0.52 | 55.2% | 0.14 |

*V31 figures are on the static SP50 universe. On the PIT dataset, V31 achieves 9.4% CAGR / Sharpe 0.54 — the gap vs SP50 is survivor bias.*

### V34–V39: point-in-time S&P 500 universe (bias-corrected)

| # | Variant | CAGR | Sharpe | MaxDD | Win rate | Rates DD |
|---|---------|------|--------|-------|----------|----------|
| 34 | PIT raw (no quality filters) | +11.48% | 0.55 | 29.3% | 35.8% | 12.8% |
| 35 | V34 + dvol top-50 screen | +8.66% | 0.63 | 24.1% | 34.6% | 15.6% |
| 36 | V35 + 10-day persistence | +9.66% | 0.71 | 22.1% | 38.5% | 13.2% |
| 37a | V36 + UVXY 15% gate-close hedge (full) | ~+7.6% | ~0.55 | ~31% | — | — |
| 37b | V36 + UVXY 15% gate-close hedge (30d cap) | ~+7.6% | ~0.55 | ~29% | — | — |
| 38 | V36 + 30-day reopen persistence waiver | ~+8.1% | ~0.60 | ~25% | — | — |
| **39** | **V36 + SPY recovery re-entry (champion)** | **+10.26%** | **0.76** | **20.5%** | **37.1%** | **13.3%** |
| R1 | V39 + RS filter (beat SPY 6mo) — see note | +10.70% | 0.78 | 18.7% | 37.3% | 15.5% |
| R2 | V39 + rank rotation (top-name-not-held, min_hold=10) — see note | +9.4% | 0.69 | 22.6% | 39% | 20% |
| R3 | V39 + fallen-out rotation (longest-held exit) — see note | +9.7% | 0.72 | 21.2% | 38.5% | 13% |
| RG1 | V39 + GLD cash substitute (idle cash → GLD) — see note | +10.0% | 0.70 | 30.9% | 53% | 22% |
| H39 | V39 signal on hindsight SP50 (ceiling) | +10.00% | **0.84** | **18.3%** | 36.1% | **12.2%** |
| — | SPY buy-and-hold | +8.10% | 0.53 | 55.2% | — | 24% |

*All figures: next_bar MOO fill (f=0/0, opening auction price), IRA/tax-free, data through 2026-04-25. V37a/b/V38 are legacy-fill estimates (no change). V34–V36 formerly used legacy fill; figures above are MOO-fill reruns. R2/R3/RG1 are from compare.py runs using the same fill model.*

**Key finding:** With the corrected MOO fill model V39 now leads H39 on CAGR (+10.26% vs +10.00%) while H39 retains the edge on Sharpe (0.84 vs 0.76) and MaxDD (18.3% vs 20.5%). V39 beats SPY by +2.16pp CAGR. **R1 (RS filter) outperforms V39 on all three headline metrics** — see closed research section. R2/R3 rotation experiments add turnover without improving risk-adjusted returns. RG1 (GLD parking) adds ~0.3pp CAGR long-run but widens MaxDD by ~10pp — not worth it.

### Rolling 5-year windows (22 windows, 2000–2026, step=12mo, IRA/tax-free)

![V39 rolling 5-year CAGR by start date](docs/rolling_5y_v39.png)

| Variant | Mean | Min | P25 | Median | P75 | Max | > SPY | > QQQ |
|---------|------|-----|-----|--------|-----|-----|-------|-------|
| **V39 (champion)** | **+8.8%** | **+2.1%** | **+5.0%** | **+8.7%** | **+12.2%** | **+16.5%** | **11/22** | **5/22** |
| V36 (reference) | +7.5% | −0.2% | +5.2% | +7.5% | +10.0% | +14.1% | 7/22 | 5/22 |
| SPY B&H | +9.1% | −2.2% | +1.9% | +11.7% | +14.9% | +18.2% | — | — |
| QQQ B&H | +11.9% | −15.7% | +5.9% | +15.5% | +19.3% | +28.1% | — | — |

V39 **best window:** 2013 start (+16.5% CAGR, α=+1.4pp vs SPY). **Worst window:** 2007 start (+2.1%, GFC) — V39 never goes negative across all 22 five-year windows. V36 dips to −0.2% in the same window. V39 beats SPY in 11/22 windows (up from 9/22 in legacy mode) because more realistic T+1 fills reduce the number of marginal entries that underperform. All figures use `next_bar` fill mode (f=0.1).

---

## What each lever does (measured)

**SMA period (50/200 vs 100/300):** Single largest return driver. +3.6pp CAGR. Captures multi-year sustained trends (NVDA, MSFT, AAPL) with longer holds. Trade-off: larger drawdowns because the slow signal stays long longer into reversals.

**Momentum filter (top 25%):** Largest single drawdown reduction. Cuts MaxDD 55% → 31% (−24pp) by avoiding laggard stocks even on valid golden-cross signals.

**SPY regime gate:** Strongest crash-protection lever. GFC DD cut from 30% → 15% by sitting flat during 2002 and 2008–2009 bears. Cost: misses first weeks of each recovery.

**VIX adaptive sizing (half at VIX>25, pause at VIX>40):** Cleanest risk reducer. Gradual scaling preserves half-sized entries during elevated fear, capturing recoveries from spikes.

**Adaptive reopen (V31):** Detects rate-driven bears via ^IRX momentum. When rates rising (20d MA > 60d MA): use slow MA-cross reopen. When easing: use fast price-cross. Eliminates 2022 gate whipsaw (54 transitions in 18 months) without regressing COVID/GFC recoveries.

**Dollar-volume screen (V35):** Top-50 by trailing 30d avg dvol. Rotates out distressed names as liquidity drains months before bankruptcy. Recovers 2pp CAGR from raw PIT baseline.

**Trend persistence gate (V36):** Requires 10 consecutive LONG days before entry. Eliminates whipsaw entries on stocks that briefly cross SMA100/300 then reverse (WFC, BAC, CAT). Lifts win rate 30.5% → 36.1%, cuts MaxDD −2.5pp.

**SPY recovery re-entry (V39):** When SPY < SMA200 but SPY > SMA20 (recovering), allows entries at 25% normal size. Captures V-shaped recoveries (GFC 2009, COVID 2020, rate shock 2022) before the full gate reopens. Tested sma ∈ {20,50,100} × mult ∈ {0.25,0.5,0.75} — sma=20, mult=0.25 dominates all axes. Adds +0.84pp CAGR, +0.07 Sharpe, −4.4pp MaxDD vs V36.

**T-bill yield:** +0.4pp mean CAGR; +2.1pp in high-rate eras. Strictly additive.

---

## Closed research (not deployed)

**RS relative-strength gate (R1):** Hypothesis — require each entry to outperform SPY over the trailing 126 days (beat SPY 6-month return) on top of dvol top-50 + momentum top-25%. Previously rejected under legacy fill mode (CAGR +8.64% vs V39 +9.24%, −0.60pp). **With corrected MOO fill model, R1 now outperforms V39 on all three headline metrics: CAGR +10.70% vs +10.26% (+0.44pp), Sharpe 0.78 vs 0.76, MaxDD 18.7% vs 20.5%.** It also beats H39 on CAGR (+10.70% vs +10.00%) and nearly matches on Sharpe (0.78 vs 0.84). Infrastructure (`rs_filter`, `rs_lookback` params) in code.

The prior rejection analysis (legacy fill mode) was misleading: legacy fills ignoring gap dynamics masked the RS filter's selective edge. With MOO fills that correctly price gap-down entries and gap-up exits, the RS filter's benefit (removing names that are in absolute uptrends but lagging SPY on a 6-month basis) becomes clearly positive. The 15.5% Rates DD (vs V39's 13.3%) is the only metric where V39 beats R1.

**Status: re-open for evaluation as V39 replacement.** Needs forward-test validation before promotion.

**UVXY gate-close hedge (V37a/b):** Hypothesis — buy UVXY when SPY regime gate closes. Result: net negative. Two problems: (1) adaptive ^IRX reopen causes daily gate flips during 2022 grinding bear → 20+ one-day entries with friction; (2) one 289-day hold (May 2022–Mar 2023) lost −74.2% from contango decay. UVXY contango (~50–80%/yr) makes it unsuitable as a prolonged bear hedge.

**Reopen persistence waiver (V38):** Hypothesis — waive 10-day persistence for 30 days after gate reopens to catch V-shaped recoveries. Result: exactly neutral. V36 and V38 identical on all metrics.

**Short strategies (V18, V26–V28):** All underperformed. SH (inverse ETF) lost $44k over 8 gate periods. Lagging signal: by the time SMA200 is breached, most of the profitable short has already occurred. Conclusion: sit in cash.

**Mean reversion (MR, 5 variants):** Best standalone result +1.76% CAGR — a drag when blended (+0.90% on its $35k slice diluted combined from +10.03% to +7.18%). Paused.

**Two-stage universe re-rank (V40):** Hypothesis — widen dvol pool to top-100, then re-rank by rolling 30-day Sharpe and take the top 50; surfaces consistent grinders over names that are merely large. Result: CAGR dropped from +9.20% to +7.68%, Sharpe from 0.69 to 0.54. Root cause: 30-day Sharpe favors recent momentum, causing the universe to rotate into names that just had a good month rather than names in confirmed SMA100/300 uptrends — two competing momentum signals. Rejected.

**R² efficiency gate (V41):** Hypothesis — within the dvol top-50 liquidity floor, rank names by rolling R² of log(price) vs linear time; R²≈1 = clean straight-line trend, low R² = volatile/meandering. Keep top X% by R² before the existing momentum filter. Swept r2_lookback ∈ {126, 252} × r2_top_pct ∈ {0.25, 0.5, 0.75} (1.0 = V39 baseline):

| lookback | top_pct | CAGR | Sharpe | MaxDD | WinRate |
|---|---|---|---|---|---|
| — | 1.0 (V39) | **+9.20%** | **0.69** | **20.1%** | 37.5% |
| 126 | 0.25 | +7.98% | 0.68 | 21.9% | **40.2%** |
| 126 | 0.50 | +8.25% | 0.67 | 24.8% | 39.6% |
| 126 | 0.75 | +8.44% | 0.67 | 25.8% | 38.5% |
| 252 | 0.50 | +8.18% | 0.68 | 21.9% | 38.5% |
| 252 | 0.75 | +7.42% | 0.61 | 22.8% | 37.4% |
| 252 | 0.25 | +6.46% | 0.59 | 25.2% | 33.9% |

126-day lookback outperforms 252-day across all pct values — shorter window more responsive to current trend quality. The gate lifts win rate (37.5% → 40.2% at best) but consistently trades CAGR for it without improving Sharpe. Root cause: R² filtering also excludes legitimately strong trending names during their early breakout phase when price is still choppy. V39 remains champion. Rejected.

**Downside RS recovery filter (V42):** Hypothesis — during SPY recovery mode (SPY > SMA20, < SMA200), restrict new entries to the top-10 tickers by "downside RS": cumulative excess return over SPY on SPY-down days over the trailing 22 trading days. Names that held up on bad days should lead the recovery. Result: CAGR dropped from +9.21% to +8.10%, Sharpe from 0.69 to 0.59, end value from $1,012,742 to $774,497. Root cause: V39's recovery-mode edge is *broad participation* in an early rally across the full dvol top-50 universe; restricting to 10 names reduces diversification and misses the broad-based snapback. The filter is correct in theory but too narrow in practice. Rejected.

**Trailing ATR ratchet stop (V43):** Hypothesis — replace the fixed `entry − N×ATR` stop with a ratchet: `stop = max(prev_stop, close − N×ATR)`. Stop trails price upward, locking in gains on climax-top reversals. Swept multiplier ∈ {2.5, 3.0, 3.5}:

| Multiplier | End $ | CAGR | Sharpe | MaxDD | Trades |
|---|---|---|---|---|---|
| — (V39, fixed 3.0×) | **$1,012,742** | **+9.21%** | **0.69** | **20.1%** | ~900 |
| trailing 2.5× | $511,530 | +6.4% | 0.53 | 23.4% | 1608 |
| trailing 3.0× | $674,972 | +7.5% | 0.60 | 23.7% | 1433 |
| trailing 3.5× | $632,486 | +7.3% | 0.59 | 20.0% | 1325 |

All three lose 1.7–2.8pp CAGR and generate 1.4–1.8× more trades. Root cause: the trailing stop exits during normal mid-trend pullbacks that the SMA100/300 signal is designed to ride through. The SMA exit is the better primary signal; a tighter initial stop (already at 3×ATR) handles catastrophic loss. Infrastructure (`trailing_stop` flag) kept in code. Rejected.

**Earnings exit filter (E1/E2/E3):** Hypothesis — on T-1 before known earnings dates, exit positions with insufficient unrealized cushion (in ATR units) to absorb an adverse gap. Three variants: E1 exit if cushion < 1.5×ATR (conservative), E2 < 2.0×ATR (aggressive), E3 exit all regardless. Data: yfinance historical earnings dates (limit=100, ≈25yr history for major S&P 500 members).

| Variant | End $ | CAGR | Sharpe | MaxDD | Earnings exits |
|---|---|---|---|---|---|
| V39 baseline | **$1,012,742** | **+9.21%** | **0.69** | **20.1%** | 0 |
| E1 (< 1.5× ATR) | $788,460 | +8.17% | 0.62 | 22.8% | 159 |
| E2 (< 2.0× ATR) | $823,211 | +8.35% | 0.63 | 23.0% | 203 |
| E3 (global) | $904,283 | +8.73% | 0.66 | 21.2% | 686 |

All three are net negative. Key insight from E3: exiting 100% of positions before every known earnings announcement costs only −0.48pp CAGR over 26 years. This confirms that stocks in confirmed SMA100/300 uptrends tend to beat earnings expectations — the strategy is inherently already selecting for positive earnings momentum. Exiting before announcements removes that embedded edge. Infrastructure (`earnings_exit_atr` param, `data/earnings.py` cache) kept in code. Rejected.

**VIX halving suspension in recovery mode (V44):** In V39, recovery-mode entries (SPY < SMA200 but SPY > SMA20, allowed at 0.25× size) are additionally halved when VIX > 25, giving 0.125× effective size. Hypothesis: floor the VIX multiplier at 1.0 during recovery mode so halving doesn't stack on the already-conservative 0.25× size. The VIX > 40 full pause is preserved.

Result (from 2022): V44 CAGR 7.81% vs V39 7.79% — negligible. The recovery window where VIX > 25 AND SPY > SMA20 is too narrow to move the needle; the sizing effect is dominated by the 0.25× recovery floor, not the VIX halving. Infrastructure (`recovery_vix_floor` param) kept in code. Rejected.

**Conditional persistence after regime reopens (V45):** Hypothesis — the 10-day trend persistence gate resets to zero when the regime reopens after a prolonged bear, blocking entries for 2 extra weeks into strong recoveries. After the 2022 bear (gate closed May 2022 – Mar 2023), the persistence gate cost ~16 fewer entries in H1 2023. Fix: for `reopen_grace_days` calendar days after a regime reopen from a closure ≥ `min_closure_days`, require only `reopen_short_persist` days (e.g., 5) instead of 10. `min_closure_days=90` was added to avoid activating the grace after short closures like COVID (~40 trading days).

Probe result (no persistence from 2022): +12.74% CAGR, Sharpe 0.75 — confirms there were profitable entries available in H2 2023 and 2024 that tighter persistence was missing. However, V45 with 5-day grace for 40 days post-reopen produced +7.21% CAGR vs V39's 7.79% — net negative.

Root cause: the March 2023 regime reopen coincided with the SVB banking crisis (March 10–17, 2023). The 40-day grace window pulled forward entries in GS, MA, MRK, PEP during peak crisis volatility — entries V39's strict 10-day filter naturally avoided. The extra losses (~$1,400 from grace-window entries) outweighed the recovery benefit. Importantly, the H2 2023 profitable entries that drove the "no persistence" probe were already captured by V39 once clean 10-day signals formed post-crisis. The SVB crisis is a single idiosyncratic event invisible in the signals — optimizing around it would be overfitting. Infrastructure (`reopen_short_persist`, `reopen_grace_days`, `min_closure_days` params) kept in code. Rejected.

**Position cap relaxation (15% → 20%/25%):** Motivated by V39 trailing SPY by ~7.7pp/yr over 2024–2026 while large AI/mega-cap names (NVDA, MSFT, META) ran hard. Wash-sale exclusions are not applied in backtesting (PIT universe includes all S&P 500 members); the cap is the binding constraint on single-name concentration.

| Cap | CAGR (2yr) | Sharpe (2yr) | CAGR (26yr) | Sharpe (26yr) |
|-----|-----------|-------------|------------|--------------|
| 15% (baseline) | +12.31% | 0.74 | +9.20% | 0.69 |
| 20% | +15.22% | 0.90 | +8.76% | 0.65 |
| 25% | +17.32% | 0.98 | +8.62% | 0.64 |

The 2-year improvement is real but purely a recent-cycle effect. Over 26 years, relaxing the cap hurts: −0.44pp CAGR and −0.58pp CAGR at 20% and 25% respectively, as higher concentration amplifies drawdowns in dotcom and GFC bear markets. The 15% cap is load-bearing across regimes. Classic overfitting trap — looks compelling in the period of concern, rejected by the full history. Rejected.

**Rank-based rotation (R2) — two implementations, both rejected:** Hypothesis — when the portfolio is missing the #1 momentum-ranked name, exit the weakest held position and bring in the top name.

*First implementation (min_hold=5, any non-held outranker):* Triggered whenever any non-held, filter-passing name outranked the weakest held position by ≥ 0.15 percentile. 2yr result: CAGR +20.7% vs V39 +23.9%, doubled turnover (146 vs 78 trades). Daily momentum rank is too noisy — the rotation churned into names that quickly reversed. Rejected.

*Second implementation (min_hold=10, top-name-not-held trigger):* Only triggers when the absolute #1 ranked name is not currently held; exit the lowest-ranked held position (held ≥ 10d). This is far more selective — only 10 rotation events in 2 years. 2yr: CAGR +24.4% (+0.5pp), Sharpe 1.24 (+0.06) — small edge. Full history (25yr): CAGR +9.4% (−0.3pp), Sharpe 0.69 (−0.03), MaxDD 22.6% (+1.4pp), 1130 trades vs 901. The full history rejects: rotation adds 229 extra trades over 25 years without converting them into returns. The 2yr edge is a recency artifact — recent market has had faster sector rotation than the long-run average. Infrastructure (`rotation_threshold`, `rotation_min_hold`, `rotation_fallen_out` params) kept in code. Rejected.

**Fallen-out momentum rotation (R3):** Hypothesis — only rotate out a held position if it has specifically dropped out of the momentum top-25% filter (was a leader, is no longer one), AND the #1 ranked name is not held. Exit the longest-held fallen-out position. 2yr result: identical to V39 (0 rotation events). Root cause: by the time a held position drops out of the top-25% momentum filter, the SMA100/300 signal has already flipped to CASH and the normal signal-exit catches it first — the window between "still LONG signal, no longer top-25%" is structurally empty in practice. Infrastructure kept. Rejected.

**GLD cash substitute (RG1):** Hypothesis — idle cash earns ~5% T-bill yield; substituting it with GLD (gold ETF, ETF20 member) gives equity-like returns on uninvested capital. GLD is removed from normal entry signals in this variant; all idle cash above a 2% buffer is deployed into GLD daily; GLD is sold on-demand to fund new long entries. `cash_yield=False` to avoid double-counting.

| Period | V39 (T-bills) | RG1 (GLD) | Delta |
|--------|--------------|-----------|-------|
| 2yr CAGR | +23.9% | +25.2% | +1.3pp |
| 2yr Sharpe | 1.18 | 1.20 | +0.02 |
| 2yr MaxDD | 17.7% | 20.6% | +2.9pp |
| Full CAGR | +9.7% | +10.0% | +0.3pp |
| Full Sharpe | **0.72** | 0.70 | −0.02 |
| Full MaxDD | **21.2%** | 30.9% | +9.7pp |

GLD adds ~0.3pp CAGR long-run but the MaxDD cost is severe (+9.7pp). The 2012–2018 GLD bear market hit idle capital hard during a period when V39 was flat-to-down in equities anyway. The 2yr edge is recency — GLD has been strong since 2022. Infrastructure (`cash_substitute_etf`, `cash_substitute_min_buffer_pct` config fields) is generic and supports any parking ETF (BIL, TLT, IAU, etc.) for future experiments. Rejected on risk-adjusted basis; infrastructure retained.

**Rolling window interpretation note:** The rolling 5-year CAGR is measured from a live continuous simulation — no rebalancing to cash at window boundaries — which is the correct comparison against SPY B&H. The worst window (+2.0%, 2007 start covering the GFC) shows V39 never going negative across all 22 windows. A cold-start run from any given year will differ from the rolling window because it begins fully in cash with no inherited positions; the rolling window represents a real investor running V39 continuously since 2000. These measure different things and should not be directly compared.

---

## Fill simulation: canonical model

All numbers in this document use `next_bar` MOO fill (f=0/0) — the correct model for pre-open limit orders on liquid S&P 500 names.

**Fill mode mechanics:**
- `legacy`: entry at close[T] × 1.001, exit at close[T] × 0.999. Same-day fill — unrealistic.
- `next_bar` f=0: entry and exit at open[T+1]. Correct for a limit order placed pre-market: the opening auction clears at a single price with no bid-ask spread. A buy limit ≤ open fills at open; a sell limit ≥ open fills at open. Gap-up buys (>1%) and gap-down sells that never touch the limit go unfilled.

| Entry f | Exit f | CAGR | Sharpe | MaxDD | Calmar | Notes |
|---|---|---|---|---|---|---|
| legacy | legacy | +9.24%† | 0.68† | 20.5% | 0.45 | idealized — same-day close fill |
| **0.0** | **0.0** | **+10.26%** | **0.76** | **20.5%** | **0.50** | **canonical — opening auction** |
| 0.1 | 0.1 | +9.72% | 0.73 | 20.2% | 0.48 | conservative haircut |
| 0.3 | 0.5 | ~+7% | ~0.55 | ~25% | — | pessimistic / intraday |

†Legacy CAGR appears lower than MOO because legacy fills ignore gap-down entry windfalls and gap-up exit windfalls; it also fills gap-up buy entries at close price (unrealistic — a buy limit is never filled above market).

**Why MOO (f=0) is the correct default:** A limit order placed after EOD before T+1 open participates in the opening auction. The auction clears at a single price — there is no bid-ask spread at the auction. If the limit is in the money at open, you fill at the open price. On liquid S&P 500 names, bid-ask spread at open is $0.01–0.05, which is noise.

```bash
python mixa/backtest/run.py --variant 4                          # next_bar MOO (default)
python mixa/backtest/run.py --variant 4 --fill-mode legacy       # legacy mode for reference
```

---

## Trend strategy conclusion: V39 is the champion

V39 is the **only fully bias-corrected variant** — no foreknowledge, all filters real-time computable, dynamic S&P 500 universe. H39 (same signal, static SP50) is the apples-to-apples hindsight ceiling.

| Metric | **V39 (bias-corrected champion)** | R1 (RS filter candidate) | H39 (hindsight ceiling) | SPY | AOA§ |
|--------|-----------------------------------|--------------------------|------------------------|-----|------|
| End value | $1,306,185 | **$1,451,775** | $1,228,219 | $775,385 | $555,686§ |
| CAGR | +10.26% | **+10.70%** | +10.00% | +8.10% | +6.74%§ |
| Sharpe | 0.76 | 0.78 | **0.84** | 0.53 | 0.77§ |
| MaxDD | 20.5% | 18.7% | **18.3%** | 55.2% | 28.4%§ |
| GFC stress | 20.2% | **18.4%** | 18.2% | 55% | — |
| COVID stress | **14.5%** | **14.5%** | 17.0% | 34% | — |
| Rates stress | **13.3%** | 15.5% | 12.2% | 24% | — |
| Win rate | 37.1% | **37.3%** | 36.1% | — | — |
| Min 5yr CAGR | **+2.1%** (2007) | — | — | −2.2% | n/a |

§AOA (iShares Core Aggressive Allocation ETF, 80/20 stocks/bonds) from inception 2008-11-11. All figures IRA/tax-free, next_bar MOO fill (f=0/0), data through 2026-04-23.

**Interpretation:** With corrected MOO fills, V39 now leads H39 on CAGR (+10.26% vs +10.00%) and COVID/Rates stress DDs; H39 retains the edge on Sharpe and MaxDD (quality universe premium). V39 beats SPY by +2.16pp CAGR and +0.24 Sharpe. **R1 (RS filter) now dominates V39 on CAGR, Sharpe, and MaxDD** — re-open for forward-test evaluation as the next champion candidate. A quality screen that closes the Sharpe gap to H39 (0.84) remains the primary research target.

![V39 vs H39 equity curve](docs/v39_vs_h39.png)

---

## Current recommended configuration: V39

- SMA100/300 golden cross on daily bars
- Dynamic PIT S&P 500 universe; forced-exit on removal
- Dollar-volume screen: top-50 by trailing 30d avg dollar volume
- Momentum filter: top 25% by 12-1 month cross-sectional rank
- 10-day trend persistence gate before entry
- SPY asymmetric regime gate: close at SMA200, reopen fast (^IRX easing) or slow (^IRX tightening)
- SPY recovery re-entry: 25% size when SPY < SMA200 but > SMA20
- VIX adaptive sizing: half-size VIX > 25; skip VIX > 40
- ATR stop: 3× ATR14 below entry; never trailed
- 20-day re-entry cooldown after stop-losses
- T-bill cash yield: idle cash earns ^IRX daily
- Position sizing: 1% portfolio risk per trade, max 15% per position
- *Full-period (2000–2026): CAGR +10.26%, Sharpe 0.76, MaxDD 20.5% (next_bar MOO fill, IRA/tax-free)*

---

## Strategy 2: Mean reversion (paused)

**Signal:**
1. RSI(2) < 10 — very short-term oversold
2. Price > SMA200 — not in a structural downtrend
3. Today's range > 1.5× average range — confirms panic, not drift

**Exit:** RSI(2) > 70 (reversion complete), or 5-day time stop, or 4% hard stop.

**Best result (MR V3):** CAGR +1.76%, Sharpe 0.44, MaxDD 28.0%, win rate 63.1%. Standalone MR does not justify deployment — it earned +0.90% CAGR on its $35k slice in the blended backtest, dragging combined CAGR from +10.03% → +7.18%.

---

## Strategy 3: Long vol hedge (not deployed)

UVXY tested as V37a/b — net negative vs V36 due to contango decay on prolonged holds. Not deployed. See closed research above.

---

## Concentration analysis (V39)

*Regenerate with `python mixa/backtest/compare.py --variant 4`.*

V36 concentration is structurally broader than V29/V31 (static SP50) because the PIT universe rotates in and out of distressed names, spreading P&L more evenly. NVDA remains the top contributor but a lower share of total P&L than in V29 (15–20% vs 30%), since the full S&P 500 universe introduces more diversified winners (energy, financials, industrials across different eras).

---

## Momentum lookback sweep (M-series)

Swept lookback ∈ {3-1, 6-1, 9-1, 12-1 months} plus a 50/50 blend of 6-1 and 12-1 against V39 baseline. All variants share one precompute (lookback is simulation-only). Turnover is complete round-trips/year.

| Variant | End $ | CAGR | Sharpe | MaxDD | Turnover | GFC | COVID | Rates |
|---|---|---|---|---|---|---|---|---|
| **M4 (12-1, V39 baseline)** | **$1,306,185** | **+10.3%** | **0.76** | **20.5%** | 34/yr | 20% | 15% | 13% |
| M3 (9-1) | $1,232,455 | +10.0% | 0.73 | 21.8% | 35/yr | 19% | 14% | 20% |
| M5 (blend 6/12) | $1,079,902 | +9.5% | 0.70 | 26.1% | 35/yr | 21% | 14% | 24% |
| M2 (6-1) | $1,042,959 | +9.3% | 0.68 | 25.8% | 35/yr | 20% | 15% | 24% |
| M1 (3-1) | $874,256 | +8.6% | 0.64 | 23.7% | 34/yr | 19% | 16% | 23% |

**12-1 month dominates on every metric** — highest CAGR (+0.3pp over 9-1), best Sharpe (+0.03), and best Rates-stress drawdown (13% vs 20–24%). Shorter lookbacks inflate MaxDD and Rates DD without improving turnover. The blend of 6-1 and 12-1 tracks 9-1 closely and offers no advantage.

**Why 12-1 wins here:** The SMA100/300 signal already filters for slow, sustained trends. Momentum at 12-1 months aligns with that timeframe — it ranks names that have been trending for a full year. Shorter lookbacks (3-1, 6-1) pick up recent mean-reversion candidates that often conflict with the SMA signal, increasing whipsaw. Rates-period performance (2022) is especially sensitive: 12-1 held 13% MaxDD vs 20–24% for shorter windows, because it avoided high-momentum-recently names that reversed hardest when rates rose.

**12-1 confirmed optimal. V39 unchanged.**

---

## Name Selection Research

### Filter lead time on distressed names

**Question:** How early do the V39 entry filters (dvol top-50 + momentum top-25%) remove a name relative to its official S&P 500 removal date?

**Method:** For each trading day, compute the set of tickers eligible under all V39 filters when the regime gate is open. For each ticker with a recorded removal date, find the last day it was eligible. Lead time = removal date − last eligible date.

Script: `research/eligible_universe.py` → `research/distress_analysis.py`

**Summary (623 removals since 2000; 32 with yfinance data available):**

| Metric | Value |
|--------|-------|
| Filter fired before removal | **100%** |
| Median lead time | **1,132 days** |
| Lead > 30 days | 94% |
| Lead > 90 days | 91% |

The dvol+momentum filters remove names with massive median lead time — the filter is a strong leading indicator of index removal, especially for distressed names with large price declines.

*Note: many historically distressed names (LEHMQ, ENRNQ, WCOEQ, WAMUQ, BSC) are excluded from this table because yfinance no longer carries their historical data. Ghost data is available for some (LEH, ENRN, WCOM) and would show even larger lead times given their multi-year pre-bankruptcy declines.*

**Removals sorted by lead time — closest calls first (tickers ever eligible under V39 filters):**

| Ticker | Removed from S&P 500 | Last eligible | Lead days | Peak→removal | Notes |
|--------|---------------------|---------------|-----------|--------------|-------|
| RIG    | 2008-12-19 | 2008-09-01 |   109 | −71% | Transocean — GFC fast collapse |
| FNMA   | 2008-09-11 | 2007-09-20 |   357 | −99% | Fannie Mae conservatorship |
| ENPH   | 2025-09-22 | 2023-05-24 |   852 | −88% | Enphase Energy collapse |
| VIAV   | 2013-12-23 | 2011-05-10 |   958 | −99% | Viavi Solutions |
| CLF    | 2014-04-02 | 2011-07-20 |   987 | −78% | Cliffs Natural Resources |
| FSLR   | 2017-03-20 | 2014-04-30 | 1,055 | −82% | First Solar collapse |
| AMD    | 2013-09-23 | 2010-08-18 | 1,132 | −92% | AMD multi-year decline |
| AAL    | 2024-09-23 | 2021-08-18 | 1,132 | −81% | American Airlines post-COVID |
| FMCC   | 2008-09-11 | 2004-08-24 | 1,479 | −99% | Freddie Mac conservatorship |
| KBH    | 2009-12-21 | 2005-09-09 | 1,564 | −83% | KB Home GFC |
| ILMN   | 2024-06-24 | 2019-07-31 | 1,790 | −79% | Post-GRAIL collapse |
| CNX    | 2016-03-04 | 2010-04-22 | 2,143 | −90% | Coal/gas distress |
| GNW    | 2015-11-18 | 2009-11-06 | 2,203 | −87% | Genworth Financial |
| CMVT   | 2007-02-01 | 2001-01-15 | 2,208 | −84% | Comverse Technology fraud |
| PRGO   | 2021-09-20 | 2015-05-20 | 2,315 | −77% | Perrigo pharma decline |
| ATI    | 2015-07-02 | 2007-03-06 | 3,040 | −71% | Allegheny Technologies |
| NBR    | 2015-03-23 | 2006-05-04 | 3,245 | −73% | Nabors energy distress |
| NOV    | 2021-09-20 | 2011-06-21 | 3,744 | −84% | Oil services collapse |
| SLM    | 2014-05-01 | 2003-04-17 | 4,032 | −52% | Sallie Mae spin-off |
| BBBY   | 2017-07-26 | 2005-02-16 | 4,543 | −78% | Bed Bath & Beyond |
| THC    | 2016-04-18 | 2002-11-26 | 4,892 | −85% | Tenet Healthcare distress |

**Interpreting the closest calls:** RIG (109 days) is the only genuine fast-crash miss — Transocean collapsed quickly in the GFC with little warning from the filters. FNMA (357 days, −99%) and the names below show the dvol screen rotating out deteriorating names years before the index acts — Freddie Mac 4 years early, VIAV 2.6 years, AMD 3 years. Corporate renames (FB→META, FI→FISV) are excluded from this table; the engine handles them in-place with no forced exit.

---

## Open research directions

1. **Asymmetric position sizing:** Scale `risk_pct` to 1.5% when SPY regime is strong AND breadth > 60%; down to 0.5% when borderline. Regime-conditional sizing rather than binary.

3. **Inversion guard:** 10Y−2Y yield curve spread as a slow-motion bear signal — tighten position cap when inverted > 3 months.

4. **MR on sector ETFs:** Test RSI(2) mean reversion on XLK/XLF/XLE etc. instead of individual stocks — less gap risk, lower correlation with trend book.
