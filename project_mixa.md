---
name: MixA System Overview
description: MixA active trading algo — architecture, champion V51.SC (research), live still on V51.R, full variant research history through dual-factor gate sweep
type: project
originSessionId: f4051d13-ab5d-4c00-9d90-4303e7d654b1
---
MixA is an active trading algo at `/home/brewadmin/projects/trading/mixa/`.
Venv: `/home/brewadmin/projects/trading/mixa/venv/`. $100k simulation capital.

**Current live runner: V51.R** (runner_eod_v51r.py, promoted 2026-05-03)
- Hold SSO when SPY regime OPEN; exit only when regime closes AND VIX rank₂₅₂ ≥ 40%
- Live runner has MAX_SUPPRESSED_DAYS=10 (V51.S logic) already wired in
- IBKR not yet connected; IBKRExecutor has known bugs

**Research champion: V51.SC** (not yet deployed — pending decision)
- V51.R + N=10 sustained-close override + two-phase MA-cross reopen guard (SMA50 Phase 2)
- Backtest 2006–26 (real SSO): CAGR 25.6%, MaxDD 29.8%
- Backtest 2000–26 (synthetic SSO): CAGR 13.1%, MaxDD 48.9%, Dot-com MaxDD 41.0%
- Cost vs V51.R: −1.9pp CAGR (2006–26). Benefit: halves tail-risk in slow-onset bears.

**Full variant progression (2000–2026, synthetic SSO):**
| Variant | CAGR | Sharpe | MaxDD | Dot-com | GFC |
|---|---|---|---|---|---|
| V50 (SPY 1×, regime only) | 9.5% | 0.86 | 31.6% | 31.6% | 13.3% |
| V51 (SSO 2×, regime only) | 12.7% | 0.62 | 47.4% | 42.8% | 32.9% |
| V51.R (+ VIX gate) | 10.8% | 0.53 | 74.4% | 74.4% | 60.9% |
| V51.SC (+ N=10 + MA-cross, SMA50 P2) | 13.1% | — | 48.9% | 41.0% | — |

Key insight: V51 (raw regime, no VIX gate) beats V51.R on every metric. VIX gate solved
a minor whipsaw problem but introduced catastrophic slow-bear suppression.

**V51.SC two-phase guard:**
- N=10 consecutive days of VIX suppression → forced exit → two-phase guard activates
- Phase 1: wait for SMA100 < SMA200 (death cross = bear confirmed)
- Phase 2: wait for SMA50 > SMA200 (golden cross = recovery confirmed)
- Normal VIX exits: fast re-entry preserved (guard only on N=10 forced exits)

**V51.SC diagnostic (2026-05-04):**
- COVID 2020: VIX rank hit 100% on exit day → VIX exit, guard never activated, zero drag
- False overrides (2006–26): 3 forced exits (2008 GFC valid; 2019-02-04 false override)
- Dot-com exposure: 146 bdays total; 128 are open-regime (gate cannot act); unavoidable
- Phase 2 sensitivity: SMA50 gives −4.6pp dot-com MaxDD vs SMA100 at similar 2006-26 CAGR

**Rolling density research (2026-05-04) — NOT adopted:**
- All 4 configs (10/15, 12/15, 14/20, 18/25) still show 146-day dot-com exposure
- 128/146 days regime is genuinely OPEN — rolling density cannot act before regime closes
- Improves dot-com MaxDD to 34.6% but causes 7 forced exits in 2006–26 (vs 3) → CAGR collapses

**Dual-factor regime gate research (2026-05-04) — NOT adopted:**
Adds early-warning close condition to SPY > SMA200. Tested: DD-6/8/10%, MACD(12,26,9), ROC-60.

| Variant | Dotcom exp | First exit | Dot-com MaxDD | 2006–26 CAGR |
|---|---|---|---|---|
| V51.SC baseline | 146 bd | 2000-10-16 (−9.6%) | 41.0% | 25.6% |
| DD-6% + V51.SC | 14 bd | 2000-04-13 (−6.1%) | 47.2% | 14.2% |
| DD-8% + V51.SC | 15 bd | 2000-04-14 (−11.4%) | 37.9% | 19.2% |
| DD-10% + V51.SC | 15 bd | 2000-04-14 (−11.4%) | 51.5% | 22.0% |
| MACD(12,26,9) | 8 bd | 2000-04-05 (−2.8%) | 29.0% | 3.2% |
| ROC-60 | 14 bd | 2000-04-13 (−6.1%) | 41.3% | 8.1% |

All candidates reduce dot-com exposure to 8–15 bdays. Root cause of 2006–26 CAGR collapse:
exits are VIX-triggered (not N=10 forced) → two-phase guard never activates → strategy
re-enters during 2001–2003 bear bounces when rolling 252-day high drifts within range.
DD-8% is best balance but still costs 6.4pp CAGR (unusable). MACD persistence is what makes
it effective (29% dot-com MaxDD) but same persistence kills normal-market returns (3.2%).

**Open research directions:**
1. Asymmetric dual-factor gate: use early-warning signal for CLOSE but stricter/different
   condition for REOPEN (e.g., require MACD > 0 AND drawdown < 8% to reopen)
2. Extend two-phase MA-cross guard to ALL exits (not just N=10 forced) — keeps strategy in
   cash through full 2001–2003 bear regardless of regime re-entry signal
3. Absolute VIX level trigger (VIX > 25 forces close regardless of regime state)

**Architecture:**
- `backtest/vR_extended_research.py` — full research script (all sweeps, diagnostics)
- `live/runner_eod_v51r.py` — EOD signal (V51.S logic deployed)
- `live/runner_morning.py` — pre-market order preview
- `state/` — SqliteStateStore (`~/.mixa/simulation.db`, reset 2026-05-03 to $100k)
- `docs/alpha_summary.md` — full research history (primary reference)
- `docs/project_mixa.md` — this file, also in repo

**Regime gate logic:**
- Close: SPY closes below SMA200
- Reopen: SPY closes above SMA100 AND ^IRX (3-month T-bill) easing or flat
- VIX rank₂₅₂ = (VIX − min₂₅₂) / (max₂₅₂ − min₂₅₂); exit threshold = 40%
