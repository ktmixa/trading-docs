# VIX Call Tail-Hedge Overlay — Strategy & Simulation Notes

**Module:** `backtest/run_vix_call_overlay.py`  
**Live layer on:** V52.DD12 (UPRO trend-following + VIX rank gate + DD12 guard)  
**Current parameters:** K=45, DTE=90, budget 1%/yr, trigger VIX≥50

---

## Strategy in plain English

While the portfolio is invested in UPRO, we spend a small fixed fraction of NAV
each month buying far-OTM VIX calls. The calls cost nearly nothing in normal times
and pay off explosively if volatility spikes. When VIX closes at or above 50 —
a level historically only seen in genuine crises — we sell everything on the next
morning's open and stop buying until the equity signal re-enters UPRO. The budget
is sized so the drag in calm years is negligible (roughly 0.08% of NAV per month)
while the payoff in a crisis can offset a meaningful fraction of the drawdown.

---

## Exact mechanics

### Budget & sizing

| Parameter | Value |
|-----------|-------|
| Annual budget | 1.0% of NAV |
| Monthly allocation | NAV × (1% / 12) = ~0.0833%/month |
| Strike | K = 45 (VIX call) |
| DTE at entry | 90 calendar days |
| Contract multiplier | ×100 |

On the first trading day of each month where state = ACCUMULATING and V52 is
in UPRO, the full monthly budget is deployed. Number of contracts:

```
n = floor(monthly_budget / (call_price × 100))
```

### State machine

```
ACCUMULATING  ──→  (VIX close ≥ 50)  ──→  PAUSED
                                              │
                         V52 re-enters UPRO  │
                         (prev=False, now=True)
                                              ↓
                                       ACCUMULATING
```

Existing options carry through re-arm — they are not liquidated when
transitioning PAUSED → ACCUMULATING; only the buying halt resets.

### Expiry handling

- **OTM (VIX < K at expiry):** contracts expire worthless; proceeds = 0.
- **ITM (VIX ≥ K at expiry):** settled at intrinsic = (VIX_spot − K) × 100 × n_contracts.
  Proceeds added back to the portfolio cash pool.

### Threshold sell

Triggered when VIX **closes** ≥ 50. Modelled as a market-on-open order the next
morning (T+1 MOO). Sell price uses next-day VIX open as the forward price input
(falls back to same-day close if next-day open is unavailable). All active
batches are liquidated in one event.

---

## Pricing model — Black model, NOT market data

**There is no historical VIX options data in this simulation.** Every price —
entry, daily MTM, and exit — is computed analytically using the **Black model**
(European call on a futures-like underlying, r=0):

```
d1 = [ln(F/K) + 0.5σ²T] / (σ√T)
d2 = d1 − σ√T
price = F·N(d1) − K·N(d2)
```

where:
- **F = VIX spot** (used as a proxy for the 90-day VIX futures price)
- **K = 45** (strike)
- **T** = remaining DTE / 365
- **σ = 1.20** (120% annualised vol, constant — the "vol of VIX")

### Entry price

```
entry_price = max(BS_price × 1.05, $0.02)
```

The 5% slippage is intended to approximate the bid-ask spread on a very cheap
OTM option. This is the only concession to real-world execution on the buy side.

### Exit price (threshold sell)

```
sell_price = BS_black(moo_vix, K=45, T_remaining, σ=1.20)
```

No slippage applied on exit. The simulation sells at the **Black model mid** using
next-day VIX open as the underlying. In practice, a MOO market order in a spike
would fill at or near the bid, not the mid.

### MTM

Daily portfolio NAV = v52_pool + Σ BS_mid(vix_close, K, T_rem, σ) × 100 × n_contracts

---

## Key approximation risks

These are the places where the simulation is likely optimistic:

| Issue | Direction | Magnitude |
|-------|-----------|-----------|
| VIX spot ≠ VIX futures | Overestimates option value when spot > front-month future (backwardation) | Moderate |
| Constant σ=120% | OTM VIX calls trade at *higher* implied vol than ATM (steep right skew); 120% may underprice OTM calls in normal times and overprice them in spikes | Unclear; likely underprices entry cost |
| Exit at BS mid, not bid | In a real VIX spike the bid-ask on OTM calls widens sharply; filling a large block at mid is optimistic | Could reduce sell proceeds 10–30% |
| No market impact on exits | Selling 88–329 contracts of far-OTM VIX calls at once in a distressed market has non-trivial market impact | Optimistic |
| No bid-ask on daily MTM | Marked at mid, so unrealised P&L is overstated before realisation | Cosmetic — only affects intra-period curve |
| Jan 2025 batch (88 contracts, expiry Apr 2) expired 6 days before the Apr 8 spike | This is a real structural limitation: 90 DTE windows don't always overlap with crises | Realistic — actually conservative in that case |

**Bottom line:** the directional result (VIX calls add meaningful tail protection
at low drag) is likely real, but the exact proceeds figures — especially the
THRESHOLD_SELL amounts — are probably 10–30% optimistic relative to actual
market execution.

---

## Parameter history

| Date | Change | Reason |
|------|--------|--------|
| initial | K=40, σ=120%, 1%/yr, DTE=90, trigger=50 | baseline |
| 2026-05-14 | K=40 → K=45 | K-strike sweep showed K=45 flips net option P&L from −$26K to +$12K (2000–2026); higher strike = cheaper options = more contracts per budget; meaningful intrinsic ($5) still available at trigger |

---

## Observed events (2000–2026 backtest, K=45)

Three trigger events where VIX ≥ 50:

| Event | VIX close | MOO VIX | Contracts | Proceeds |
|-------|-----------|---------|-----------|---------|
| COVID (Mar 2020) | 82.7 | ~75 | ~180 | largest |
| GFC (Oct 2008) | ~80 | — | ~60 | moderate |
| 2025 tariff (Apr 2025) | ~52 | ~45 | ~88* | smallest |

*Jan 2025 batch (88 contracts, expiry Apr 2) expired 6 days before the Apr 8
sell trigger — those contracts were gone before the spike was monetised.

---

## What would be needed for a more accurate simulation

1. **Historical VIX options chain data** (e.g. from CBOE or OptionMetrics) —
   actual mid/bid/ask for the specific strikes and expiries bought each month.
2. **VIX futures prices** rather than VIX spot as the forward proxy — especially
   important in backwardation (pre-spike) and contango (normal).
3. **Implied vol surface** for VIX options — the right skew means 45-strike calls
   in normal times trade well above what σ=120% flat vol implies.
4. **Realistic exit fills** — bid price for large block liquidations, not mid.

Until that data is available, treat the option P&L numbers as order-of-magnitude
estimates with a likely 10–30% optimism bias on the exit side.
