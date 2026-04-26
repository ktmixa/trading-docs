# V39 Sanity Check Baselines

Reference numbers for regression testing. If a code change causes these to shift
materially, investigate before proceeding.

All numbers use `next_bar` fill mode (entry-factor 0.1, exit-factor 0.1) — the
operational canonical. Legacy numbers preserved for comparison.

Stop-loss exits fill at the intraday stop price (or open if gapped through), not
at T+1 open. Signal exits fill at T+1 open ± exit_factor × range.

**Why pinned dates matter:** Open positions are marked to their last close, so
a run without a fixed `--to-date` will produce different end-value and CAGR every
day as prices move. Always pass `--to-date` and `--from-date` to reproduce exactly.
Use the most recent completed trading day as the canonical end date; update the
baseline (and this file) whenever the sanity check is intentionally re-run.

**Current canonical end date: 2026-04-23**

---

## Reproducibility limitations

These numbers are **not fully reproducible** on a fresh clone. Three reasons:

1. **L1 OHLCV pickle is gitignored.** `mixa/data/.cache/` is excluded from the
   repo (large binary, ~190 MB). A fresh clone must rebuild it by fetching from
   yfinance/Polygon. yfinance applies split and dividend adjustments retroactively,
   so prices fetched in the future will differ slightly from prices fetched today —
   every adjusted close shifts, which moves signals, stops, and P&L.

2. **Python package versions are loosely pinned.** `requirements.txt` uses `>=`
   bounds. yfinance in particular has had breaking behavior changes between minor
   versions. The installed versions that produced these numbers are recorded below.

3. **SP500 PIT dataset updates over time.** `mixa/data/sp500_ticker_start_end.csv`
   is committed but reflects index composition as of the fetch date. Running after
   future index changes will include different tickers.

**To check when the current pickle was fetched:**
```bash
python mixa/data/cache.py --info
# Shows: ohlcv_2026-04-22.pkl  (fetched 2026-04-23T...)
```
The `fetched_at` timestamp is embedded in the pickle itself as of 2026-04-23.
Pickles built before that date show "not stamped".

**Package versions used for the baselines recorded here:**
```
pandas==2.3.3
numpy==2.4.4
yfinance==1.2.1
scipy==1.17.1
requests==2.33.1
pandas_market_calendars==5.3.2
```

---

## Full history (2000-01-01 → 2026-04-23)

```bash
# Canonical (next_bar f=0.1)
python mixa/backtest/run.py --variant 4 --fill-mode next_bar --entry-factor 0.1 --exit-factor 0.1 --from-date 2000-01-01 --to-date 2026-04-23 --no-progress

# Legacy (close ±0.1% slippage)
python mixa/backtest/compare.py --variant 4 --to-date 2026-04-23 --no-report
```

| Metric | next_bar (canonical) | legacy (reference) |
|--------|---------------------|--------------------|
| End value | $848,205 | $1,022,914 |
| CAGR | +8.47% | +9.24% |
| Sharpe | 0.64 | 0.68 |
| Max DD | 23.7% | 20.5% |
| Calmar | 0.36 | 0.45 |
| Win rate | 35.0% | 35.5% |
| Trades | 909 | 916 |
| vs SPY CAGR | +0.37pp | +1.14pp |
| vs SPY Sharpe | +0.11 | +0.15 |

Recorded: 2026-04-24

---

## Last 2 years (2024-04-23 → 2026-04-23)

```bash
# Canonical (next_bar f=0.1)
python mixa/backtest/run.py --variant 4 --fill-mode next_bar --entry-factor 0.1 --exit-factor 0.1 --from-date 2024-04-23 --to-date 2026-04-23 --no-progress

# Legacy
python mixa/backtest/run.py --variant 4 --from-date 2024-04-23 --to-date 2026-04-23 --no-progress
```

| Metric | next_bar (canonical) | legacy (reference) |
|--------|---------------------|--------------------|
| CAGR | +17.82% | +21.28% |
| Sharpe | 0.91 | 1.06 |
| Max DD | 19.0% | 18.2% |
| Win rate | 32.4% | 36.6% |
| Trades | 74 | 71 |
| vs SPY CAGR | −1.99pp | +1.47pp |

**Context:** V39 next_bar underperforms SPY by −1.99pp over this window (SPY +19.81%)
due to the 2024–2025 AI/mega-cap rally; T+1 open fills on gap-up entries penalise
the V39 execution vs legacy same-day fills. Legacy mode beats SPY by +1.47pp.

Exit split (next_bar): 58 signal / 16 stop-loss

Recorded: 2026-04-24

---

## How to use

Run either command above (with the pinned dates) and compare to the table.
Acceptable drift: ±0.05pp CAGR, ±0.02 Sharpe — this can only happen if you
re-run against the same `--to-date` but with different code. Any drift on an
identical date + identical code is a regression.

A larger shift (>0.2pp CAGR or >0.05 Sharpe) after a code change indicates a
logic regression — check engine/daily.py, signals/, or precompute.

**When to update this file:**
- After intentional refactors (e.g. data source change, fill logic change)
- When advancing the canonical end date to a new trading day
- Never update just because the market moved — that's not a regression
