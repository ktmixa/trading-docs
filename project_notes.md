# MixA — Active Trading Algo

Trend-following strategy on a $100k capital pool. Current champion: **R1_189** (V39 + 9-month relative-strength filter).

| Metric | **R1_189** (champion) | V39 (prev champion) | H39 (hindsight ceiling) | SPY B&H | AOA† |
|--------|----------------------|---------------------|------------------------|---------|------|
| End value ($100k start) | **$1,078,696** | $1,016,190 | — | $781,394 | $559,722† |
| CAGR | **+9.46%** | +9.21% | — | +8.10% | +6.76%† |
| Sharpe | **0.70** | 0.69 | — | 0.53 | 0.77† |
| Max drawdown | **20.4%** | 20.4% | — | 55.2% | 28.4%† |
| Rates stress DD | 17.5% | **15.1%** | — | — | — |
| Win rate | **37.4%** | 36.2% | — | — | — |

H39 = identical V39 signal on a static hand-picked SP50 (hindsight-biased). R1_189 beats V39 by +25bp CAGR and +0.01 Sharpe with identical MaxDD. R1_189 beats SPY by +1.36pp CAGR and +0.17 Sharpe. *All figures: standard limit order fill (pre-market limit at signal_px), IRA/tax-free, data through 2026-04-26.*

†AOA (iShares Core Aggressive Allocation ETF, 80/20 stocks/bonds) from inception 2008-11-11; all others from 2000-01-01.

![R1_189 vs V39 vs SPY equity curve](docs/r1_189_vs_h39.png)

### Rolling 5-year windows (22 windows, 2000–2026, step=12mo)

*Rolling window chart reflects V39; R1_189 rolling analysis pending.*

![V39 rolling 5-year CAGR by start date](docs/rolling_5y_v39.png)

| Variant | Mean | Min | P25 | Median | P75 | Max | > SPY | > QQQ |
|---------|------|-----|-----|--------|-----|-----|-------|-------|
| **V39 (reference)** | **+8.8%** | **+2.1%** | **+5.0%** | **+8.7%** | **+12.2%** | **+16.5%** | **11/22** | **5/22** |
| V36 (reference) | +7.5% | −0.2% | +5.2% | +7.5% | +10.0% | +14.1% | 7/22 | 5/22 |
| SPY B&H | +9.1% | −2.2% | +1.9% | +11.7% | +14.9% | +18.2% | — | — |
| QQQ B&H | +11.9% | −15.7% | +5.9% | +15.5% | +19.3% | +28.1% | — | — |

Worst window: **2007 start** (+2.1%, GFC; V39 never goes negative across all 22 windows). Best: **2013 start** (+16.5%, α=+1.4pp vs SPY). V39's recovery re-entry feature keeps the minimum positive in every 5-year window.

Interactive Brokers (paper → live). Completely separate from juicer — different broker account, different DB, no shared state.

---

## What R1_189 is

**SMA100/300 golden cross** on the dynamic S&P 500 universe (point-in-time membership from the fja05680/sp500 dataset), with:
- **PIT universe gate** — only enters tickers that were actually S&P 500 members on the entry date; force-exits on removal (bankruptcy, downgrade, M&A)
- **Wash-sale exclusions** — `WASH_SALE_EXCLUDE` in `config.py` (`GOOG`, `GOOGL`, `AMZN`, `META`, `DDOG`) removed from the universe; these are held in juicer and opening/closing within 30 days of a juicer assignment would disallow the loss
- **Dollar-volume screen** — top-50 by trailing 30-day avg dollar volume; rotates out distressed names as liquidity drains months before bankruptcy
- **Momentum filter** — only enter top 25% by 12-1 month cross-sectional rank
- **RS filter** — entry requires trailing 189-day return > SPY trailing 189-day return; ensures entries are market leaders, not just stocks in absolute uptrends (new in R1_189 vs V39)
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

| Variant | Universe | CAGR | Sharpe | MaxDD | Win rate |
|---------|----------|------|--------|-------|----------|
| V34 | PIT raw | +11.48% | 0.55 | 29.3% | 35.8% |
| V35 | V34 + dvol top-50 | +8.66% | 0.63 | 24.1% | 34.6% |
| V36 | V35 + 10-day persistence | +9.66% | 0.71 | 22.1% | 38.5% |
| V39 | V36 + SPY recovery re-entry | +9.21% | 0.69 | 20.4% | 36.2% |
| **R1_189** | **V39 + RS filter (9mo)** | **+9.46%** | **0.70** | **20.4%** | **37.4%** |
| H39 | V39 signal, hindsight SP50 | — | — | — | — |

*Standard limit order fill (pre-market limit at signal_px), IRA/tax-free, data through 2026-04-26.*

The dollar-volume screen recovers CAGR by rotating out distressed names before collapse (ENRN, LEH, WCOM exited top-50 months before zero). The persistence gate lifts win rate and cuts MaxDD. The recovery re-entry adds CAGR by capturing V-shaped recoveries. The RS filter (R1_189) adds +25bp CAGR without expanding MaxDD — it filters out stocks in absolute uptrends that are nonetheless lagging the market (dead money). H39 is the apples-to-apples hindsight ceiling; not yet rerun under the current fill model.

---

## Quick start

```bash
cd $HOME/projects/trading/algo
source venv/bin/activate

# ── Single variant (full metrics + trade detail + concentration report) ───────
python mixa/backtest/compare.py --variant 7          # R1_189 champion

# ── All variants side-by-side ─────────────────────────────────────────────────
python mixa/backtest/compare.py

# ── List all 32 variants with indices ─────────────────────────────────────────
python mixa/backtest/compare.py --list

# ── Tax profiles ──────────────────────────────────────────────────────────────
python mixa/backtest/compare.py --variant 7 --tax-profile nyc   # NYC bracket
python mixa/backtest/compare.py --variant 7 --tax-profile ira   # IRA (default)
# options: ira, nyc, ca, fl, tx  |  or --tax-short 0.35 --tax-long 0.20

# ── Rolling 5-year window robustness ─────────────────────────────────────────
python mixa/backtest/rolling.py
python mixa/backtest/rolling.py --window 3      # 3-year windows
python mixa/backtest/rolling.py --plot          # save chart to plots/

# ── EOD signal runner — simulation account (no broker required) ───────────────
python mixa/live/runner_eod.py                          # simulation mode (default)
python mixa/live/runner_eod.py --mode simulation        # explicit
python mixa/live/runner_eod.py --mode paper             # paper IBKR account
python mixa/live/runner_eod.py --verbose                # with signal details
python mixa/live/scheduler.py --now                     # same, via scheduler wrapper
python mixa/live/scheduler.py                           # run daily at 4:20pm ET

# ── Morning runner ────────────────────────────────────────────────────────────
python mixa/live/runner_morning.py                      # simulation: preview only
python mixa/live/runner_morning.py --mode paper         # submit to IBKR paper
python mixa/live/runner_morning.py --mode live          # submit to IBKR live
python mixa/live/runner_morning.py --dry-run            # preview without submitting

# ── Account management ────────────────────────────────────────────────────────
python mixa/live/manage.py --status --mode simulation   # cash, positions, pending orders
python mixa/live/manage.py --status --mode paper
python mixa/live/manage.py --reset  --mode simulation   # wipe to $100k, no positions
```

---

## Fill simulation

`BacktestExecutor` uses `next_bar` MOO fill by default — the correct model for pre-open limit orders. **All headline numbers above use this model.**

| Mode | Entry fill | Exit fill | CAGR (R1_189) | Sharpe | MaxDD |
|------|-----------|-----------|---------------|--------|-------|
| **`next_bar` f=0 (default)** | open[T+1] | open[T+1] | **+9.46%** | **0.70** | **20.4%** |
| `next_bar` f=0.1 | open + 0.1×(high−open) | open − 0.1×(open−low) | ~+9.0% | ~0.67 | ~20% |
| `legacy` | close × 1.001 | close × 0.999 | n/a† | — | — |

A limit order placed pre-market participates in the opening auction, which clears at a single price with no bid-ask spread. f=0 (MOO) is the correct default. Legacy fills are unrealistic (fills gap-up buy entries at close price, ignoring that a buy limit is never filled above market) and kept only for historical reference.

```bash
python mixa/backtest/run.py --variant 7                       # next_bar MOO (default)
python mixa/backtest/run.py --variant 7 --fill-mode legacy    # legacy mode (reference only)
```

†Legacy no longer tracked against headline metrics; standard limit order model is canonical as of 2026-04-26.

---

## Debugging: run.py flags

`run.py` is the low-level single-run runner. Use it to isolate a specific variant, date window, or ticker.

```bash
# Run variant 7 (R1_189) over full history
python mixa/backtest/run.py --variant 7

# Custom date window (YYYY-MM-DD or shorthands: 1y, 6mo, 30d)
python mixa/backtest/run.py --variant 7 --from-date 5y
python mixa/backtest/run.py --variant 7 --from-date 2007-06-01 --to-date 2009-12-31

# Named stress scenarios (sets from/to and plot title in one flag)
python mixa/backtest/run.py --variant 7 --scenario gfc
# options: full, dotcom, gfc, covid, rates, since-dotcom, since-gfc, since-covid, since-rates

# Debug a single ticker (overrides the entire universe)
python mixa/backtest/run.py --variant 7 --ticker NVDA
python mixa/backtest/run.py --variant 7 --ticker BAC --from-date 5y

# --detail: print a line for every position open and close
python mixa/backtest/run.py --variant 7 --detail
python mixa/backtest/run.py --variant 7 --ticker WFC --detail  # trace one whipsaw

# --no-progress: suppress the rolling equity/DD lines during simulation
python mixa/backtest/run.py --variant 7 --no-progress

# Combine for focused debugging: single ticker, date window, full trade trace
python mixa/backtest/run.py --variant 7 --ticker BAC --from-date 2022-01-01 --detail --no-progress

# Plot equity curve
python mixa/backtest/run.py --variant 7 --plot
python mixa/backtest/run.py --variant 7 --scenario gfc --plot-save /tmp/gfc.png
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

32 variants in `backtest/variants_trend.py`. All use the dynamic PIT S&P 500 universe. Run `--list` to see indices.

| # | Label | Type | Description | Status |
|---|-------|------|-------------|--------|
| 1 | V34 | StrategyConfig | PIT baseline — no quality filters | reference |
| 2 | V35 | StrategyConfig | V34 + trailing dollar-volume top-50 screen | reference |
| 3 | V36 | StrategyConfig | V35 + 10-day trend persistence | reference |
| 4 | V39 | StrategyConfig | V36 + SPY recovery re-entry | prev champion |
| 5 | R1 | StrategyConfig | V39 + RS filter (beat SPY 6mo, 126d) | closed — 9mo lookback superior |
| 6 | R1_63 | StrategyConfig | V39 + RS filter (beat SPY 3mo, 63d) | reference |
| **7** | **R1_189** | **StrategyConfig** | **V39 + RS filter (beat SPY 9mo, 189d)** | **champion** |
| 8 | R1_252 | StrategyConfig | V39 + RS filter (beat SPY 12mo, 252d) | reference |
| 9 | R1_QQQ | StrategyConfig | V39 + RS filter (beat QQQ 6mo) | closed — QQQ too restrictive |
| 10 | R1_RSP | StrategyConfig | V39 + RS filter (beat RSP equal-weight 6mo) | closed — RSP too restrictive |
| 11 | R1_MOO | BacktestConfig | R1 + MOO fills (fills always at open) | reference — fill model comparison |
| 12 | R2 | StrategyConfig | V39 + rank rotation (top-name-not-held, min_hold=10) | closed — no long-run edge |
| 13 | R3 | StrategyConfig | V39 + fallen-out rotation (longest-held exit) | closed — never fires |
| 14 | RG1 | StrategyConfig | V39 + GLD cash substitute (idle cash → GLD) | closed — MaxDD +9.7pp |
| 15 | V44 | StrategyConfig | V39 + no VIX halving in recovery mode | closed — negligible |
| 16 | V45 | StrategyConfig | V39 + conditional persistence (5d reopen grace) | closed — net negative |
| 17 | H39 | StrategyConfig | V39 signal on hindsight SP50 (ceiling) | reference ceiling |
| 18–22 | M1–M5 | StrategyConfig | V39 momentum lookback sweep (3-1 / 6-1 / 9-1 / 12-1 / blend) | reference — 12-1 optimal |
| 23–32 | LP_* | StrategyConfig | Limit-price offset sweep (2yr window, entry/exit ±0.5%/1.0%) | closed — 0% optimal |

Closed research (in git history):
- **V1–V33** — static SP50 universe (hindsight bias); documented in `docs/alpha_summary.md`
- **V37a/b** — UVXY 15% hedge on gate close; net negative (contango decay)
- **V38** — 30-day reopen persistence waiver; exactly neutral

**Infrastructure features** (available via `StrategyConfig` fields, inactive by default):
- `rotation_threshold / rotation_min_hold / rotation_fallen_out` — rank-based rotation (R2/R3)
- `cash_substitute_etf / cash_substitute_min_buffer_pct` — parking ETF for idle cash (RG1); swap to BIL, TLT, IAU etc. by changing the ticker
- `rs_filter / rs_lookback / rs_benchmark` — relative-strength filter vs benchmark (R1 series)
- `trailing_stop` — ATR ratchet stop (tested as V43, net negative)
- `earnings_exit_atr` — pre-earnings exit on cushion threshold (net negative)
- `entry_limit_pct / exit_limit_pct` — limit price offsets (LP sweep confirms 0.0% optimal for both)

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
├── docs/project_notes.md     <- This file; project overview and file structure
├── config.py                 <- BACKTEST_START, WASH_SALE_EXCLUDE, risk limits, tax rates
├── compute.py                <- Shared precomputation (signals, dvol, regime, VIX, etc.)
│                                Used by both backtest runner and live EOD runner.
├── backtest/
│   ├── run.py                <- Single-run backtest CLI (--variant, --detail, --ticker, --scenario)
│   │                            Defines: StrategyConfig, SimulatorConfig, BacktestConfig, run_backtest()
│   ├── precompute.py         <- Shim: re-exports from compute.py (backward compatibility)
│   ├── params.py             <- V36 canonical params + Monte Carlo SEARCH_SPACE
│   ├── runner.py             <- run_suite(): groups variants by precompute key; MC helpers
│   ├── compare.py            <- Multi-variant comparison CLI (--variant, --list, --tax-profile)
│   ├── compare_mr.py         <- MR variant comparison CLI
│   ├── rolling.py            <- Rolling 5-year window analysis (V36/V39 vs SPY/QQQ)
│   ├── plot_contenders.py    <- Equity chart + IRX robustness report
│   ├── run_blended.py        <- Archived blended portfolio runner (V16 era)
│   ├── variants_trend.py     <- Active trend variant definitions (32 variants)
│   └── variants_mr.py        <- 5 MR variant definitions
├── engine/
│   └── daily.py              <- DailyEngine: per-day strategy logic (shared by backtest + live)
├── executor/
│   ├── base.py               <- AbstractExecutor, OrderRequest, OrderResult
│   ├── backtest.py           <- BacktestExecutor (simulates fills using T+1 OHLCV)
│   ├── deferred.py           <- DeferredExecutor: queues orders to SQLite; no broker needed (EOD)
│   ├── ibkr.py               <- IBKRExecutor (ib_insync; paper + live via IB Gateway)
│   └── approval_gate.py      <- Human-approval wrapper (optional gate before order submission)
├── state/
│   ├── store.py              <- StateStore ABC, MemoryStateStore, SqliteStateStore
│   └── schema.sql            <- SQLite schema (positions, trades, pending_orders, snapshots, etc.)
├── live/
│   ├── runner_eod.py         <- Phase 1: EOD signals → pending orders in SQLite (no broker)
│   │                            --mode simulation|paper|live drives which DB is used
│   │                            Simulation mode: runs fill simulation at start of each run
│   ├── runner_morning.py     <- Phase 2: simulation=preview, paper/live=submit to IBKR
│   ├── manage.py             <- Account management CLI (--reset, --status per mode)
│   ├── runner.py             <- All-in-one runner (connects IBKR; use after paper trading is live)
│   ├── scheduler.py          <- NYSE-aware scheduler: fires runner_eod.py at 4:20pm ET
│   ├── market.py             <- load_live_ohlcv(), is_market_day(), LOOKBACK_DAYS=550
│   └── reconcile.py          <- IBKR vs StateStore reconciliation (PARTIAL/GHOST/UNTRACKED)
├── reports/
│   └── daily.py              <- generate() for live runs; generate_eod() for EOD signal sheets
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
    ├── r1_189_vs_h39.png        <- R1_189 vs V39 equity curve (2000–present)
    ├── v39_vs_spy.png           <- V39 vs SPY equity curve (2000–present, archived)
    ├── rolling_5y_v39.png       <- Rolling 5-year CAGR chart (V39 vs V36 vs SPY/QQQ)
    ├── contenders_equity.png    <- V29/V31/V33 vs SPY/QQQ equity curve (archived)
    └── reports/
        ├── eod/                 <- EOD signal sheets (YYYY-MM-DD.md, one per trading day)
        ├── backtest/            <- Backtest run reports ({run_name}.md)
        └── paper/               <- Paper/live run reports ({YYYY-MM-DD}_{run_name}.md)
```

---

## Code architecture

### Type hierarchy

```
StrategyConfig          — All strategy parameters: signals, filters, sizing, universe, tax.
                          Used directly by the live EOD runner (runner_eod.py).
                          31 of 32 VARIANTS are StrategyConfig instances.

SimulatorConfig         — Backtest-only execution parameters: fill_mode, starting_capital.
                          Documents what BacktestConfig adds over StrategyConfig.
                          Replaced by IBKRConfig in live trading.

BacktestConfig          — StrategyConfig + SimulatorConfig fields (inheritance).
(StrategyConfig)          Used by compare.py / run.py for the one variant that needs
                          non-default fill_mode (R1_MOO, fill_mode='moo').
                          getattr fallbacks handle StrategyConfig instances transparently.
```

### Three executors

| Executor | Fill model | Used by |
|----------|-----------|---------|
| `BacktestExecutor` | Simulates fills using T+1 OHLCV (next_bar or legacy) | backtest/run.py, compare.py |
| `DeferredExecutor` | Queues orders to SQLite; no broker connection required | runner_eod.py (all modes) |
| `IBKRExecutor` | Submits limit orders to IB Gateway (paper port 4002, live 4001) | runner_morning.py (paper/live) |

All three implement `AbstractExecutor` (executor/base.py). The engine never inspects which executor is in use — `DeferredExecutor` is swapped for `IBKRExecutor` at the morning runner level once IB Gateway is connected.

### Four account modes and databases

Each mode is fully isolated — separate SQLite file, separate position state, separate cash balance.

| Mode | DB path | Position source of truth | Fill confirmation |
|------|---------|--------------------------|-------------------|
| `backtest` | in-memory only | `MemoryStateStore` (thrown away after run) | `BacktestExecutor` (T+1 bar) |
| `simulation` | `~/.mixa/simulation.db` | SQLite — updated by EOD fill simulation | `_simulate_pending_fills()` at next EOD |
| `paper` | `~/.mixa/paper.db` | IBKR paper account; SQLite is cache | `IBKRExecutor` (morning runner) |
| `live` | `~/.mixa/live.db` | IBKR live account; SQLite is cache | `IBKRExecutor` (morning runner) |

**Simulation** runs the full signal pipeline without a broker. At the start of each EOD run, yesterday's pending orders are replayed against today's actual OHLCV bar using `compute_limit_fill()` — the same logic as the backtest. Confirmed fills update the positions table and cash balance. Unfilled orders (gap-ups, etc.) are simply dropped.

**Paper / live** will reconcile SQLite against the real IBKR account at each EOD run start (via `live/reconcile.py`) once IB Gateway is wired up. IBKR is always the source of truth; SQLite is a local cache for fast position reads.

### Shared computation path

`compute.py` (top-level) is imported by both paths. The same precomputed signals (SMA100/300, dvol ranks, regime flags, VIX, momentum ranks, RS ratios) and the same `engine/daily.py` strategy logic run in backtest and live. No duplicate implementations.

```
Backtest:    backtest/run.py  → compute.precompute() → DailyEngine → BacktestExecutor
Simulation:  runner_eod.py   → _simulate_pending_fills() → compute.precompute() → DailyEngine → DeferredExecutor
Paper/live:  runner_eod.py   → reconcile(IBKR) → compute.precompute() → DailyEngine → DeferredExecutor
                                                          ↓ next morning
                                               runner_morning.py → IBKRExecutor
```

`backtest/precompute.py` is a backward-compat shim that re-exports from `compute.py`.

---

## Daily execution flow

The live runner is split into two phases so signal computation (cheap, no broker) is cleanly separated from order submission (requires IBKR).

### Simulation mode (current — no broker)

```
4:20 PM ET — runner_eod.py --mode simulation  (cron)
  ├── load OHLCV via yfinance / Polygon
  ├── [NEW] _simulate_pending_fills(): replay yesterday's pending orders against
  │         today's OHLCV bar using compute_limit_fill() — same logic as backtest.
  │         Filled buys → save_position() in simulation.db; fills → subtract cash.
  │         Filled sells → close_position(); cash returned. Unfilled → dropped.
  ├── load confirmed positions + cash from simulation.db (post fill-sim)
  ├── compute signals via compute.precompute() (SMA100/300, regime, dvol, momentum, RS)
  ├── run DailyEngine.step() with DeferredExecutor  [strategy: R1_189, VARIANTS[6]]
  ├── save pending orders → ~/.mixa/simulation.db (pending_orders table)
  └── write docs/reports/eod/YYYY-MM-DD.md → push to GitHub

8:30 AM ET — runner_morning.py --mode simulation
  └── prints pending orders for review (no submission — fills happen at next EOD)
```

### Paper / live mode (once IBKR is connected)

```
4:20 PM ET — runner_eod.py --mode paper|live  (cron)
  ├── load OHLCV
  ├── reconcile SQLite cache against IBKR actual positions (live/reconcile.py)
  ├── load confirmed positions + cash (from reconciled SQLite)
  ├── compute signals + run engine step
  └── save pending orders → ~/.mixa/paper.db or live.db

8:30 AM ET — runner_morning.py --mode paper|live
  ├── read pending_orders from DB
  ├── connect to IBKR (port 4002 paper / 4001 live)
  ├── submit each order as pre-market limit at signal_px (prior close)
  ├── log fills → orders audit table
  └── clear pending_orders
```

**Current status:** Simulation mode is live — EOD runner is scheduled and writing reports to `docs/reports/eod/`. Simulation DB is at `~/.mixa/simulation.db`, initialised at $100,000 cash. Morning runner for simulation is display-only. IBKR not yet connected.

The backtest (`backtest/run.py`) uses `BacktestExecutor` and is completely independent of this flow. Re-run any time:
```bash
python mixa/backtest/run.py --variant 7               # R1_189, full history
python mixa/backtest/run.py --variant 7 --scenario gfc # GFC stress
```

---

## Account management

`live/manage.py` is the CLI for inspecting and resetting account state. It is mode-aware — each mode has its own isolated database.

### Status

```bash
python mixa/live/manage.py --status --mode simulation
python mixa/live/manage.py --status --mode paper
python mixa/live/manage.py --status --mode live
```

Prints: cash balance, open positions (ticker, shares, entry price, stop, date), pending orders, cooldowns, regime gate state.

### Reset

```bash
python mixa/live/manage.py --reset --mode simulation   # interactive confirm required
```

Clears positions, pending orders, cooldowns, regime state, and daily snapshots. Sets cash back to $100,000. Keeps the trades and order audit log (permanent history).

**Only available for simulation.** Paper and live accounts are owned by IBKR — positions there can only be managed via the IBKR portal or TWS. Running `--reset` on paper/live prints `NOT IMPLEMENTED` with a TODO to pull account state from the broker API.

### Cash tracking

Cash in the simulation DB reflects confirmed fills only (updated by `_simulate_pending_fills()` at each EOD run). It is not updated by the engine step itself — deferred orders are pending, not confirmed. On T0 (first run or after reset) cash starts at $100,000.

For paper/live, cash will be read from the IBKR account at reconcile time once the broker is connected.

### DB locations

| Mode | DB path | Notes |
|------|---------|-------|
| simulation | `~/.mixa/simulation.db` | Self-contained; no broker needed |
| paper | `~/.mixa/paper.db` | Requires IBKR paper (port 4002) |
| live | `~/.mixa/live.db` | Requires IBKR live (port 4001) |
| backtest | in-memory | No file; MemoryStateStore |

Pass `--db /path/to/custom.db` to any runner to override the default path.

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
