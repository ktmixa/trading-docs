---
name: MixA System Overview
description: MixA active trading algo — architecture, new candidate V51.DD12, prior champion V51.SC, live on V51.R, full variant history through exit-DD threshold guard research
type: project
originSessionId: f4051d13-ab5d-4c00-9d90-4303e7d654b1
---
MixA is an active trading algo at `/home/brewadmin/projects/trading/mixa/`.
Venv: `/home/brewadmin/projects/trading/mixa/venv/`. $100k simulation capital.

**Current live runner: V51.R** (runner_eod_v51r.py, promoted 2026-05-03)
- Hold SSO when SPY regime OPEN; exit only when regime closes AND VIX rank₂₅₂ ≥ 40%
- Live runner has MAX_SUPPRESSED_DAYS=10 (V51.S logic) already wired in
- IBKR not yet connected; IBKRExecutor has known bugs

**New research candidate: V51.DD12** (not yet deployed — pending decision)
- V51.R gate + exit-DD threshold guard at 12% (no N=10, no MA-cross guard)
- On any exit where trailing DD from 252-day high > 12%: arm reopen guard
- Reopen guard: MACD(12,26,9) histogram > 0 AND DD < 8% (both simultaneously)
- Shallow exits (DD ≤ 12%): fast re-entry preserved
- Backtest 2006–26 (real SSO): CAGR 24.9%, MaxDD 33.3%
- Backtest 2000–26 (synthetic SSO): CAGR 12.3%, MaxDD 39.4%, Dot-com MaxDD 32.4%
- vs V51.SC: −0.7pp CAGR, −8.6pp dot-com MaxDD, −1.4pp GFC MaxDD, no false overrides

**Prior research champion: V51.SC** (superseded by V51.DD12 as candidate)
- V51.R + N=10 sustained-close override + two-phase MA-cross reopen guard (SMA50 Phase 2)
- Backtest 2006–26 (real SSO): CAGR 25.6%, MaxDD 29.8%
- Backtest 2000–26 (synthetic SSO): CAGR 13.1%, MaxDD 48.9%, Dot-com MaxDD 41.0%

**Full variant progression (2000–2026, synthetic SSO):**
| Variant | CAGR | MaxDD | Dot-com | GFC | 2006-26 CAGR |
|---|---|---|---|---|---|
| V50 (SPY 1×, regime only) | 9.5% | 31.6% | 31.6% | 13.3% | — |
| V51 (SSO 2×, regime only) | 12.7% | 47.4% | 42.8% | 32.9% | — |
| V51.R (+ VIX gate) | 10.8% | 74.4% | 74.4% | 60.9% | 27.4% |
| V51.SC (+ N=10 + MA-cross) | 13.1% | 48.9% | 41.0% | 33.4% | 25.6% |
| **V51.DD12 (+ exit-DD guard 12%)** | **12.3%** | **39.4%** | **32.4%** | **32.0%** | **24.9%** |

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

**Asymmetric gate research (2026-05-04) — NOT adopted:**
V51.A: Close = SPY < SMA200 OR DD > 8%; Reopen = SPY > SMA100 AND MACD > 0 AND DD < 8%.
VIX gate neutralises early close — VIX rank ≈ 2% in 2000 suppresses exit indefinitely without
N=10. First exit still October 2000 (same as V51.SC). Asymmetric reopen does block 2001–2003
bear bounces (dot-com MaxDD 28.7% vs 41.0%), but costs 8.6pp CAGR (25.6% → 17.0%).
V51.AC (N=10 + MACD+DD guard on forced exits only): guard never activates for dot-com because
the exit was a VIX exit, not N=10 forced. Dot-com MaxDD 39.1% (1.9pp gain), CAGR −1.7pp.
Root cause confirmed: dot-com 2001–2003 re-entry protection requires guarding ALL exits, not
just forced ones. Guard must be universal or conditional on DD-at-exit depth.

**Exit-DD threshold guard sweep (2026-05-04) — V51.DD12 adopted as candidate:**
GFC first exit DD = −6.0% (Nov 2007); dot-com exit DD = −9.6% (Oct 2000).
Threshold 8%: arms for dot-com, not GFC → 28.7% dot-com MaxDD but −5.3pp CAGR.
Threshold 12%: doesn't arm on initial dot-com exit (-9.6% < 12%), but arms on deeper
2001-2002 exits → 32.4% dot-com MaxDD at only −0.7pp CAGR cost. GFC also improves.
V51.DD12 selected: best cost-efficiency seen across all guard research.

**Open research directions:**
1. V51.DD12 live runner deployment: update runner_eod_v51r.py with exit-DD threshold guard
2. Combine V51.DD12 with N=10 to close the one-re-entry gap in dot-com 2001
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
