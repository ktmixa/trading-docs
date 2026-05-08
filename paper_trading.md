# MixA Paper Trading Log — V52.DD.OTM

**Strategy:** V52.DD12 (UPRO + VIX rank gate + DD guard + MACD reopen) + Nuclear Bunker XSP put overlay  
**Mode:** IBKR Paper account (`mixairaalgo`, clientId=3, port 4002)  
**Starting NAV:** ~$1,000,000 (IBKR paper default)  
**DB:** `~/.mixa/paper.db`  
**Logs:** `logs/eod_paper.log`, `logs/morning_paper.log`

---

## Setup Notes

- Gateway shared with penny (port 4002); mixa uses clientId=3
- IBC upgraded to 3.23.0 on 2026-05-07 to fix Read-Only API mode
- Both simulation and paper modes run in parallel (simulation unchanged)
- Crontab: EOD at 17:35 ET, morning at 9:40 ET (staggered 5 min after simulation)

---

## Active Position

| Date | Ticker | Shares | Entry Price | Stop | Status |
|------|--------|--------|-------------|------|--------|
| — | — | — | — | — | Flat (awaiting first fill) |

---

## Options Position

| Date | Instrument | Expiry | Strike | Qty | Entry | Status |
|------|-----------|--------|--------|-----|-------|--------|
| — | — | — | — | — | — | None yet |

---

## Trade Log

### 2026-05-07 — Paper trading launched

**EOD Runner (17:35 ET)**
- Signal: UPRO (regime OPEN, VIX rank 20.5%, DD −0.3%, MACD +1.36, guard clear)
- IBKR sync: 0 UPRO shares (flat, day 1)
- Pending order written: **BUY 735 UPRO @ $137.38 limit** (DAY, submit before 9:28 AM)
- IBKR submit: failed — gateway in Read-Only mode (fixed same evening)
- Options action: ENTER — morning runner will compute and submit XSP put live

**Infrastructure fixes applied this day:**
- IBC upgraded 3.20.0 → 3.23.0 (fixes Read-Only API handling for IB Gateway 1046)
- `ReadOnlyApi=no` restored in `mixa.ini`
- Gateway verified accepting orders (Error 321 cleared)

**Tomorrow (2026-05-08) morning runner at 9:40 ET will:**
1. Submit pending BUY 735 UPRO @ $137.38 (marketable limit)
2. ENTER XSP put overlay: ~90 DTE (Aug expiry), budget ~$164

---

## Daily Signal Summary

| Date | Signal | VIX rank | DD | MACD | Action | UPRO px |
|------|--------|----------|----|------|--------|---------|
| 2026-05-07 | UPRO | 20.5% | −0.3% | +1.36 | BUY pending | $136.02 |

---

## Notes

- First paper run: no prior position or DB state; EOD correctly generated BUY from flat
- Morning runner submits equity + options; EOD runner generates signal + pending orders
- Reconciliation: EOD syncs IBKR positions before building orders (prevents duplicate buys)

---

## Open To-Dos

- [ ] **Get XSP options permissions on IBKR paper account** — call/chat IBKR support, ask to enable index options (XSP) on the `mixairaalgo` paper account. Once enabled, switch `INSTRUMENT` back from `'SPY'` to `'XSP'` in `live/runner_morning.py`. SPY puts are physically settled (American style) and can create a short SPY position at expiry if ITM — not allowed in an IRA. XSP is cash-settled (European style) and is the correct instrument for IRA deployment.
