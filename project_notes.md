# MixA — Active Trading Algo

Trend-following strategy on a $100k capital pool. Single-strategy deployment as of V39.

| Metric | **V39** (champion) | H39 (hindsight ceiling) | SPY B&H | QQQ B&H | AOA† |
|--------|-----------------------------------|------------------------|---------|---------|------|
| End value ($100k start) | **$1,306,185** | $1,228,219 | $775,385 | $815,014 | $555,686† |
| CAGR | **+10.26%** | +10.00% | +8.10% | +8.30% | +6.74%† |
| Sharpe | 0.76 | **0.84** | 0.53 | 0.46 | 0.77† |
| Max drawdown | 20.5% | **18.3%** | 55.2% | 83.0% | 28.4%† |
| Rates stress DD | 13.3% | **12.2%** | — | — | — |
| Win rate | **37.1%** | 36.1% | — | — | — |

H39 = identical V39 signal on a static hand-picked SP50 (hindsight-biased). With corrected MOO fills, V39 now leads H39 on CAGR (+10.26% vs +10.00%) and end value; H39 retains the edge on Sharpe (0.84 vs 0.76) and MaxDD (18.3% vs 20.5%). V39 beats SPY by +2.16pp CAGR. **R1 (RS filter variant) now outperforms V39 on all three headline metrics** — see `docs/alpha_summary.md`.

†AOA (iShares Core Aggressive Allocation ETF, 80/20 stocks/bonds) from inception 2008-11-11; V39 and SPY/QQQ from 2000-01-01. IRA/tax-free, next_bar MOO fill (f=0/0, opening auction), data through 2026-04-25.

![V39 vs H39 vs SPY — $100k from 2000](docs/v39_vs_h39.png)

![V39 vs SPY — $100k from 2000](docs/v39_vs_spy.png)

![V39 vs AOA — $100k from 2008-11-11](docs/v39_vs_aoa.png)

### Rolling 5-year windows (22 windows, 2000–2026, step=12mo)

![V39 rolling 5-year CAGR by start date](docs/rolling_5y_v39.png)

| Variant | Mean | Min | P25 | Median | P75 | Max | > SPY | > QQQ |
|---------|------|-----|-----|--------|-----|-----|-------|-------|
| **V39 (champion)** | **+8.8%** | **+2.1%** | **+5.0%** | **+8.7%** | **+12.2%** | **+16.5%** | **11/22** | **5/22** |
| V36 (reference) | +7.5% | −0.2% | +5.2% | +7.5% | +10.0% | +14.1% | 7/22 | 5/22 |
| SPY B&H | +9.1% | −2.2% | +1.9% | +11.7% | +14.9% | +18.2% | — | — |
| QQQ B&H | +11.9% | −15.7% | +5.9% | +15.5% | +19.3% | +28.1% | — | — |

Worst window: **2007 start** (+2.1%, GFC; V39 never goes negative across all 22 windows). Best: **2013 start** (+16.5%, α=+1.4pp vs SPY). V39's recovery re-entry feature keeps the minimum positive in every 5-year window.

Interactive Brokers (paper → live). Completely separate from juicer — different broker account, different DB, no shared state.

---

## What V39 is

**SMA100/300 golden cross** on the dynamic S&P 500 universe (point-in-time membership from the fja05680/sp500 dataset), with:
- **PIT universe gate** — only enters tickers that were actually S&P 500 members on the entry date; force-exits on removal (bankruptcy, downgrade, M&A)
- **Wash-sale exclusions** — `WASH_SALE_EXCLUDE` in `config.py` (`GOOG`, `GOOGL`, `AMZN`, `META`, `DDOG`) removed from the universe; these are held in juicer and opening/closing within 30 days of a juicer assignment would disallow the loss
- **Dollar-volume screen** — top-50 by trailing 30-day avg dollar volume; rotates out distressed names as liquidity drains months before bankruptcy
- **Momentum filter** — only enter top 25% by 12-1 month cross-sectional rank
- **SPY adaptive asymmetric regime gate** — halt entries when SPY < SMA200; reopen using fast mode (price > SMA100) when rates are easing, slow mode (SMA100 > SMA200) when tightening
- **^IRX rate signal** — 20d MA < 60d MA = easing (fast reopen); fires in every Fed hiking cycle
- **VIX adaptive sizing** — half-size when VIX > 25; skip entirely when VIX > 40
- **10-day trend persistence** — requires 10 consecutive LONG signal days before opening; eliminates marginal-breakout whipsaws (BAC, WFC, CAT)
- **ATR stop-loss** — static 3×ATR14 below entry price; never trailed
- **20-day re-entry cooldown** after stop-losses
- **T-bill cash yield** — idle cash earns the 13-week T-bill rate (^IRX) daily
- **SPY recovery re-entry** — when SPY < SMA200 but SPY > SMA20, allows entries at 25% normal size; captures V-shaped recoveries (GFC 2009, COVID 2020, rate shock 2022) before the full regime gate reopens

Position sizing: 1% portfolio risk per trade, max 15% per position.

---

## Survivor-bias correction: what it cost

| Variant | Universe | CAGR | Sharpe | MaxDD | Win rate | Rates DD |
|---------|----------|------|--------|-------|----------|----------|
| V34 | PIT raw | +11.48% | 0.55 | 29.3% | 35.8% | 12.8% |
| V35 | V34 + dvol top-50 | +8.66% | 0.63 | 24.1% | 34.6% | 15.6% |
| V36 | V35 + 10-day persistence | +9.66% | 0.71 | 22.1% | 38.5% | 13.2% |
| **V39** | **V36 + SPY recovery re-entry** | **+10.26%** | **0.76** | **20.5%** | **37.1%** | 13.3% |
| H39 | V39 signal, hindsight SP50 | +10.00% | **0.84** | **18.3%** | 36.1% | **12.2%** |

The dollar-volume screen recovers CAGR by rotating out distressed names before collapse (ENRN, LEH, WCOM exited top-50 months before zero). The persistence gate lifts win rate and cuts MaxDD. The recovery re-entry adds CAGR and cuts MaxDD by capturing V-shaped recoveries. H39 shows that a quality screen's edge is in *risk metrics* — H39 wins on Sharpe and MaxDD while the larger PIT pool finds more cyclical winners, giving V39 a CAGR lead.

---

## Quick start

```bash
cd /path/to/trading/algo
source venv/bin/activate

# ── Single variant (full metrics + trade detail + concentration report) ───────
python mixa/backtest/compare.py --variant 4          # V39 champion

# ── All three variants side-by-side ──────────────────────────────────────────
python mixa/backtest/compare.py

# ── Tax profiles ──────────────────────────────────────────────────────────────
python mixa/backtest/compare.py --variant 4 --tax-profile nyc   # NYC bracket
python mixa/backtest/compare.py --variant 4 --tax-profile ira   # IRA (default)
# options: ira, nyc, ca, fl, tx  |  or --tax-short 0.35 --tax-long 0.20

# ── Rolling 5-year window robustness ─────────────────────────────────────────
python mixa/backtest/rolling.py
python mixa/backtest/rolling.py --window 3      # 3-year windows
python mixa/backtest/rolling.py --plot          # save chart to plots/

# ── EOD signal runner (no broker required) ────────────────────────────────────
python mixa/live/runner_eod.py                    # compute today's orders
python mixa/live/runner_eod.py --verbose          # with signal details
python mixa/live/scheduler.py --now               # same, via scheduler wrapper
python mixa/live/scheduler.py                     # run daily at 4:20pm ET (cron also installed)

# ── Morning execution (requires IB Gateway on paper: port 4002, live: port 4001)
python mixa/live/runner_morning.py --dry-run      # preview pending orders
python mixa/live/runner_morning.py                # submit to IBKR
```

---

## Fill simulation

`BacktestExecutor` uses `next_bar` MOO fill by default — the correct model for pre-open limit orders. **All headline numbers above use this model.**

| Mode | Entry fill | Exit fill | CAGR | Sharpe | MaxDD |
|------|-----------|-----------|------|--------|-------|
| **`next_bar` f=0 (default)** | open[T+1] | open[T+1] | **+10.26%** | **0.76** | **20.5%** |
| `next_bar` f=0.1 | open + 0.1×(high−open) | open − 0.1×(open−low) | +9.72% | 0.73 | 20.2% |
| `legacy` | close × 1.001 | close × 0.999 | +9.24%† | 0.68† | 20.5% |

A limit order placed pre-market participates in the opening auction, which clears at a single price with no bid-ask spread. f=0 (MOO) is the correct default; f=0.1 is a conservative haircut. Legacy fills are unrealistic (fills gap-up buy entries at close price, ignoring that a buy limit is never filled above market).

```bash
python mixa/backtest/run.py --variant 4                       # next_bar MOO (default)
python mixa/backtest/run.py --variant 4 --fill-mode legacy    # legacy mode (reference only)
```

†Legacy CAGR lower because it prices gap-down entry windfalls as close+0.1% rather than the cheaper open price; also inflated by unrealistic gap-up buy fills.

---

## Debugging: run.py flags

`run.py` is the low-level single-run runner. Use it to isolate a specific variant, date window, or ticker.

```bash
# Run variant 4 (V39) over full history
python mixa/backtest/run.py --variant 4

# Custom date window (YYYY-MM-DD or shorthands: 1y, 6mo, 30d)
python mixa/backtest/run.py --variant 4 --from-date 5y
python mixa/backtest/run.py --variant 4 --from-date 2007-06-01 --to-date 2009-12-31

# Named stress scenarios (sets from/to and plot title in one flag)
python mixa/backtest/run.py --variant 4 --scenario gfc
# options: full, dotcom, gfc, covid, rates, since-dotcom, since-gfc, since-covid, since-rates

# Debug a single ticker (overrides the entire universe)
python mixa/backtest/run.py --variant 4 --ticker NVDA
python mixa/backtest/run.py --variant 4 --ticker BAC --from-date 5y

# --detail: print a line for every position open and close
python mixa/backtest/run.py --variant 4 --detail
python mixa/backtest/run.py --variant 4 --ticker WFC --detail  # trace one whipsaw

# --no-progress: suppress the rolling equity/DD lines during simulation
python mixa/backtest/run.py --variant 4 --no-progress

# Combine for focused debugging: single ticker, date window, full trade trace
python mixa/backtest/run.py --variant 4 --ticker BAC --from-date 2022-01-01 --detail --no-progress

# Plot equity curve
python mixa/backtest/run.py --variant 4 --plot
python mixa/backtest/run.py --variant 4 --scenario gfc --plot-save /tmp/gfc.png
```

**What each flag prints:**

| Flag | Effect |
|------|--------|
| *(default)* | Summary metrics: CAGR, Sharpe, MaxDD, Calmar, win rate, stress DDs, trade P&L breakdown (winning / losing / net), tax summary |
| `--detail` | Every entry and exit line during simulation + top/bottom 5 P&L by ticker + exit reason breakdown |
| `--no-progress` | Suppresses rolling progress ticker (cleaner output for CI / log files) |
| `--ticker TICKER` | Loads only that one ticker — fast, no PIT load, full trace in `--detail` |
| compare.py `--variant N` | `print_metrics(detail=True)` + `print_concentration_report()` — sector/ticker P&L concentration |

---

## Variants

16 variants in `backtest/variants_trend.py`. All use the dynamic PIT S&P 500 universe. Run `--list` to see indices.

| # | Label | Description | Status |
|---|-------|-------------|--------|
| 1 | V34 | PIT baseline — no quality filters | reference |
| 2 | V35 | V34 + trailing dollar-volume top-50 screen | reference |
| 3 | V36 | V35 + 10-day trend persistence | reference |
| 4 | **V39** | **V36 + SPY recovery re-entry** | **champion** |
| 5 | R1 | V39 + RS filter (beat SPY 6mo) | candidate |
| 6 | R2 | V39 + rank rotation (top-name-not-held, min_hold=10) | closed — no long-run edge |
| 7 | R3 | V39 + fallen-out rotation (longest-held exit) | closed — never fires |
| 8 | RG1 | V39 + GLD cash substitute (idle cash → GLD) | closed — MaxDD +10pp |
| 9 | V44 | V39 + no VIX halving in recovery mode | closed — negligible |
| 10 | V45 | V39 + conditional persistence (5d reopen grace) | closed — net negative |
| 11 | H39 | V39 signal on hindsight SP50 (ceiling) | reference ceiling |
| 12–16 | M1–M5 | V39 momentum lookback sweep (3-1 / 6-1 / 9-1 / 12-1 / blend) | reference — 12-1 optimal |

Closed research (in git history):
- **V1–V33** — static SP50 universe (hindsight bias); documented in `docs/alpha_summary.md`
- **V37a/b** — UVXY 15% hedge on gate close; net negative (−0.3pp CAGR vs V36)
- **V38** — 30-day reopen persistence waiver; exactly neutral

**Infrastructure features** (available via `BacktestConfig` fields, inactive in V39):
- `rotation_threshold / rotation_min_hold / rotation_fallen_out` — rank-based rotation (R2/R3)
- `cash_substitute_etf / cash_substitute_min_buffer_pct` — parking ETF for idle cash (RG1); swap to BIL, TLT, IAU etc. by changing the ticker
- `rs_filter / rs_lookback` — relative-strength filter vs SPY (R1)
- `trailing_stop` — ATR ratchet stop (tested as V43, net negative)
- `earnings_exit_atr` — pre-earnings exit on cushion threshold (net negative)

---

## Robustness: ^IRX adaptive gate signal

Fires in every genuine Fed hiking cycle, not just 2022:

| Period | IRX mode at bottom | vs baseline |
|--------|-------------------|-------------|
| Dotcom (Oct 2002) | FAST ✓ | −1.8pp (noise in ZIRP transition) |
| GFC (Mar 2009) | SLOW | −0.2pp (ZIRP edge noise) |
| COVID (Mar 2020) | FAST ✓ | negligible |
| Rate shock (Oct 2022) | SLOW ✓ | **+7.3pp** in 2022, **+6.9pp** in 2023 |

Active 41% of backtest period.

---

## What is NOT deployed

**Mean reversion (MR):** Researched across 5 variants. Best standalone result was +1.76% CAGR — a drag when blended. Paused until the signal improves.

**Long vol hedge (UVXY / V37):** V37a (unlimited hold) and V37b (30-day cap) both net negative vs V36. Not deployed.

**Short strategies (V18, V26–V28):** All rejected. Inverse ETFs (SH) lost $44k over 26 years. Conclusion: sit in cash when the SPY gate is closed.

---

## File structure

```
mixa/
├── docs/alpha_summary.md     <- Full research log (V1–V45), conclusions
├── docs/project_notes.md    <- This file; project overview and file structure
├── config.py                 <- BACKTEST_START, WASH_SALE_EXCLUDE, risk limits, tax rates
├── backtest/
│   ├── run.py                <- Single-run backtest CLI (--variant, --detail, --ticker, --scenario)
│   ├── precompute.py         <- Shared precomputation (signals, dvol, regime, VIX, etc.)
│   ├── params.py             <- V36 canonical params + Monte Carlo SEARCH_SPACE
│   ├── runner.py             <- run_suite(): groups variants by precompute key; MC helpers
│   ├── compare.py            <- Multi-variant comparison CLI (--variant, --list, --tax-profile)
│   ├── compare_mr.py         <- MR variant comparison CLI
│   ├── rolling.py            <- Rolling 5-year window analysis (V36/V39 vs SPY/QQQ)
│   ├── plot_contenders.py    <- Equity chart + IRX robustness report
│   ├── run_blended.py        <- Archived blended portfolio runner (V16 era)
│   ├── variants_trend.py     <- Active trend variant definitions (V34/V35/V36/V39 + M-series)
│   └── variants_mr.py        <- 5 MR variant definitions
├── engine/
│   └── daily.py              <- DailyEngine: per-day strategy logic (shared by backtest + live)
├── executor/
│   ├── base.py               <- AbstractExecutor, OrderRequest, OrderResult
│   ├── backtest.py           <- BacktestExecutor (legacy close-px fills or next_bar intraday model)
│   ├── deferred.py           <- DeferredExecutor: queues orders to SQLite; no broker needed (EOD)
│   ├── ibkr.py               <- IBKRExecutor (ib_insync; paper + live via IB Gateway)
│   └── approval_gate.py      <- Human-approval wrapper (optional gate before order submission)
├── state/
│   ├── store.py              <- StateStore ABC, MemoryStateStore, SqliteStateStore
│   └── schema.sql            <- SQLite schema (positions, trades, pending_orders, snapshots, etc.)
├── live/
│   ├── runner_eod.py         <- Phase 1: EOD signals → pending orders in SQLite (no broker)
│   ├── runner_morning.py     <- Phase 2: submit pending orders to IBKR after open
│   ├── runner.py             <- All-in-one runner (connects IBKR; use after paper trading is live)
│   ├── scheduler.py          <- NYSE-aware scheduler: fires runner_eod.py at 4:20pm ET
│   ├── market.py             <- load_live_ohlcv(), is_market_day(), LOOKBACK_DAYS=550
│   └── reconcile.py          <- IBKR vs StateStore reconciliation (PARTIAL/GHOST/UNTRACKED)
├── reports/
│   └── daily.py              <- generate() for live runs; generate_eod() for signal sheets
├── strategy/
│   └── v39/
│       └── reports/
│           ├── backtest/     <- Backtest run reports ({run_name}.md)
│           └── paper/        <- Paper/live run reports ({YYYY-MM-DD}_{run_name}.md)
├── data/
│   ├── market.py             <- OHLCV + VIX + T-bill rate (yfinance / SQLite cache)
│   ├── sp500_history.py      <- Point-in-time S&P 500 membership (fja05680/sp500 dataset)
│   ├── ghost_fetcher.py      <- OHLCV loader for delisted tickers (CSV → yfinance)
│   ├── ghost_data/           <- Synthetic OHLCV for ENRN, LEH, WCOM, BSC, WAMU
│   ├── sp500_ticker_start_end.csv <- Cached PIT membership (1,246 rows, 1996–2026)
│   └── etf_universe.py       <- ETF20 ticker list (regime + benchmark passthrough)
├── signals/
│   ├── trend_daily.py        <- SMA crossover signal, require_crossover param
│   ├── regime.py             <- SPY adaptive asymmetric regime gate
│   ├── momentum.py           <- Cross-sectional 12-1 month momentum rank
│   └── sector_rotation.py    <- Sector ETF momentum ranking
└── docs/
    ├── v39_vs_spy.png           <- V39 vs SPY equity curve (2000–present)
    ├── v39_vs_aoa.png           <- V39 vs AOA equity curve (2008–present)
    ├── rolling_5y_v39.png       <- Rolling 5-year CAGR chart (V39 vs V36 vs SPY/QQQ)
    ├── contenders_equity.png    <- V29/V31/V33 vs SPY/QQQ equity curve (archived)
    └── reports/
        ├── eod/                 <- EOD signal sheets (YYYY-MM-DD.md, one per trading day)
        ├── backtest/            <- Backtest run reports ({run_name}.md)
        └── paper/               <- Paper/live run reports ({YYYY-MM-DD}_{run_name}.md)
```

---

## Daily execution flow

The live runner is split into two phases so signal computation (cheap, no broker) is cleanly separated from order submission (requires IBKR).

```
4:20 PM ET — runner_eod.py (cron, no broker)
  ├── load OHLCV via yfinance / Polygon
  ├── compute signals (SMA100/300, regime, dvol, momentum)
  ├── run DailyEngine.step() with DeferredExecutor
  ├── save pending orders → ~/.mixa/state.db (pending_orders table)
  └── write strategy/v39/reports/eod/YYYY-MM-DD.md → push to GitHub

9:35 AM ET next morning — runner_morning.py (manual or scheduled)
  ├── read pending_orders from state.db
  ├── connect to IBKR
  ├── submit each order as limit near prior close
  ├── log fills → orders audit table
  └── clear pending_orders
```

**Current status:** EOD runner is scheduled and writing reports. Morning runner is wired but IBKR is not yet connected — no orders will be submitted until `runner_morning.py` is run manually.

The backtest (`backtest/run.py`) uses `BacktestExecutor` and is completely independent of this flow. Re-run any time:
```bash
python mixa/backtest/run.py --variant 4               # V39, full history
python mixa/backtest/run.py --variant 4 --scenario gfc # GFC stress
```

---

## Broker

Interactive Brokers via `ib_insync`. Requires IB Gateway running locally (headless Linux: IB Gateway + IBC + Xvfb).

| Port | Mode |
|------|------|
| 4002 | IB Gateway paper |
| 4001 | IB Gateway live |
| 7497 | TWS paper |
| 7496 | TWS live |

Set `IBKR_PORT` in `.env`. Default is `4002` (IB Gateway paper). Paper trading precedes live.

### Pre-live checklist

- [ ] IB Gateway + IBC + Xvfb running headless on the trading box
- [ ] Paper trading dry-run clean (`python3 mixa/live/runner.py --dry-run --verbose`)
- [ ] At least 2 weeks paper trading with no unexpected force-exits or sizing errors
- [ ] **TODO: Verify ticker rename handling end-to-end in live trading.** The engine
      renames positions in-place when a corporate rename occurs (e.g. FB→META), keyed
      by `TICKER_RENAMES` in `config.py`. In the live account IBKR converts automatically,
      so the engine must detect the rename, skip any order submission, and update its
      internal state to the new ticker. Confirm this path works under `IBKRExecutor`
      before the first real trade involving a rename-eligible name.
- [ ] **TODO: Handle stock-deal acquisitions where both sides are held (e.g. AET acquired by CVS).**
      In a stock deal, IBKR auto-converts target shares → acquirer shares + cash at the exchange
      ratio. Two failure modes if both sides are held simultaneously:
      (1) *Redundant sell*: engine submits a sell order for the target on PIT force-exit, but IBKR
          has already converted those shares into acquirer stock — the sell would hit the inflated
          acquirer position, not the (now-gone) target.
      (2) *Position inflation*: the acquirer position silently grows by the conversion shares. The
          engine's internal state (shares, stop, risk budget) stays at the original size, so the
          excess shares have no stop and won't be exited on signal — they are stranded.
      In the 26-year backtest, the momentum filter naturally prevented simultaneous holdings for
      every known deal pair (acquirer underperforms during deal uncertainty; target locks near deal
      price). Zero overlaps found. But this is not guaranteed.
      Fix before live: add a `CORPORATE_ACTIONS` registry keyed by (target, close_date) →
      (acquirer, exchange_ratio). On target force-exit, if acquirer also held: skip the sell,
      recompute acquirer share count = original + converted, resize to strategy target weight,
      sell any excess immediately.
- [ ] `WASH_SALE_EXCLUDE` up to date with current juicer positions
- [ ] `TICKER_RENAMES` in `config.py` current with any pending corporate actions

---

## Tax context

Owner is in the NYC highest combined bracket (~54.8%). All backtests default to **IRA (tax-free)** to show the strategy's intrinsic performance. Pass `--tax-profile nyc` to see after-tax results.
