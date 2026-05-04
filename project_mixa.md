---
name: MixA System Overview
description: MixA active trading algo — architecture, champion V51.SC (research), live still on V51.R, full variant research history
type: project
originSessionId: f4051d13-ab5d-4c00-9d90-4303e7d654b1
---
MixA is an active trading algo at `/home/brewadmin/projects/trading/mixa/`.
Venv: `/home/brewadmin/projects/trading/mixa/venv/`. $100k simulation capital.

**Current live runner: V51.R** (runner_eod_v51r.py, promoted 2026-05-03)
- Hold SSO when SPY regime OPEN; exit only when regime closes AND VIX rank₂₅₂ ≥ 40%
- 1-day debounce on base regime signal; re-entry never filtered
- Live runner has MAX_SUPPRESSED_DAYS=10 (V51.S logic) already wired in
- IBKR not yet connected; IBKRExecutor has known bugs

**Research champion: V51.SC** (not yet deployed to live runner — pending decision)
- V51.R + N=10 sustained-close override + two-phase MA-cross reopen guard
- Backtest 2006–26 (real SSO): CAGR 25.5%, MaxDD 29.8%, Calmar 0.86
- Backtest 2000–26 (synthetic SSO): CAGR 12.2%, MaxDD 48.9%, Dot-com MaxDD 45.6%
- Cost vs V51.R: −1.9pp CAGR (2006–26). Benefit: halves tail-risk in slow-onset bears.

**Full variant progression (2000–2026, synthetic SSO):**
| Variant | CAGR | Sharpe | MaxDD | Dot-com | GFC |
|---|---|---|---|---|---|
| SPY B&H | 8.2% | 0.53 | 55.2% | 47.5% | 55.2% |
| SSO B&H (2×) | 9.1% | 0.43 | 88.2% | 78.7% | 88.2% |
| V50 (SPY 1×, regime only) | 9.5% | 0.86 | 31.6% | 31.6% | 13.3% |
| V51 (SSO 2×, regime only) | 12.7% | 0.62 | 47.4% | 42.8% | 32.9% |
| V51.R (+ VIX gate) | 10.8% | 0.53 | 74.4% | 74.4% | 60.9% |
| V51.SC (+ N=10 + MA-cross) | 12.2% | 0.61 | 48.9% | 45.6% | 29.2% |

Key insight: V51 (raw regime, no VIX gate) outperforms V51.R in dot-com. The VIX gate was
added to reduce whipsaws but suppresses exits indefinitely in slow complacent bears.
V51.SC partially restores V51's properties at the cost of slightly more complexity.

**V51.SC two-phase guard mechanics:**
- Forced exit triggers after N=10 consecutive days of VIX-gate suppression
- After forced exit: Phase 1 = wait for SMA100 < SMA200 (death cross / bear confirmed)
- After Phase 1: Phase 2 = wait for SMA100 > SMA200 (golden cross / recovery confirmed)
- Normal VIX-triggered exits are unaffected (fast re-entry preserved for GFC/COVID)

**V51.SC diagnostic results (2026-05-04):**
- COVID 2020: VIX rank hit 100% on exit day → VIX exit, no guard activated, zero drag
- False overrides (2006–26): 3 forced exits (2008 GFC valid, 2015 borderline, 2019-02-04 false)
  - 2019-02-04 false override is the primary source of −1.9pp CAGR drag
- Dot-com Phase 1 exposure: 146 bdays (March 24 → October 16 2000); 128 of those are
  open-regime days — gate cannot act; remaining 18 bdays end via VIX crossing 40%
- Phase 2 sensitivity: SMA50 gives −4.6pp dot-com MaxDD (41.0% vs 45.6%), +4.2pp GFC DD

**Rolling density research (2026-05-04) — NOT adopted:**
- Hypothesis: replace consecutive N=10 counter with "X closed days out of Y-day window"
  to survive 1-day regime bounces that reset the consecutive counter
- Result: all 4 configs (10/15, 12/15, 14/20, 18/25) still show 146-day dot-com exposure
- Root cause: 128 of 146 days the regime is genuinely OPEN (gate cannot act at all)
- RD does improve dot-com MaxDD to 34.6% (via SMA50 Phase 2), but causes 7 forced exits
  in 2006–26 vs 3 for baseline → 2006-26 CAGR collapses to 17–21%
- Conclusion: boiling-frog fix must happen at the regime signal level, not the gate

**Open research directions:**
1. Faster regime close signal (SMA150 or SMA100 instead of SMA200)
2. Peak-based trailing stop (close if SPY falls X% from 252-day high near SMA200)
3. Absolute VIX level trigger (VIX > 25 forces close regardless of regime state)

**Cron schedule (ET):**
- 4:20 PM weekdays: runner_eod_v51r.py --mode simulation
- 8:30 AM weekdays: runner_morning.py --mode simulation

**Architecture:**
- `data/`      — OHLCV cache + yfinance fetches
- `backtest/`  — run.py (regime engine), vR_extended_research.py (full research history)
- `live/`      — runner_eod_v51r.py, runner_morning.py
- `state/`     — SqliteStateStore (~/.mixa/simulation.db, reset 2026-05-03 to $100k)
- `docs/`      — alpha_summary.md (full research history)

**Regime gate logic (shared):**
- Close: SPY closes below SMA200
- Reopen: SPY closes above SMA100 AND ^IRX (3-month T-bill) easing or flat

**VIX rank₂₅₂ (min-max):** (VIX − min₂₅₂) / (max₂₅₂ − min₂₅₂)
Exit threshold = 40%. Rank gives "crisis memory" (GFC spike stays in denominator for months).
