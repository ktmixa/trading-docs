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
| 2026-05-08 | UPRO | 735 | $137.38 limit | — | Submitted (DAY, orderId=6) |

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

---

### 2026-05-08 — Morning runner + strike step fix

**Morning Runner (9:40 ET)**
- UPRO equity: BUY 735 @ $137.38 limit submitted (orderId=6, status=Submitted) ✓
- Options: XSP P 515.5 Aug-21 rejected — Error 200, no security definition
  - Root cause: `MIN_STRIKE_STEP = 0.50` produced strike 515.5; XSP uses $5 increments at 30% OTM (valid nearby: 510, 515, 520, 525)
  - Permissions were fine all along — XSP options visible, 198 contracts listed for Aug-21
  - Fix: `MIN_STRIKE_STEP = 5.0` → strike 515.0, qualifies cleanly (conId=853518536)

**Post-market fix (same day)**
- XSP options permissions confirmed working via `reqContractDetails` (198 contracts, Aug-21)
- `MIN_STRIKE_STEP` corrected to 5.0; XSP P 515.0 Aug-21 qualified successfully ✓
- Runner ready for full dual-leg execution tomorrow morning

---

## Daily Signal Summary

| Date | Signal | VIX rank | DD | MACD | Action | UPRO px |
|------|--------|----------|----|------|--------|---------|
| 2026-05-07 | UPRO | 20.5% | −0.3% | +1.36 | BUY pending | $136.02 |
| 2026-05-08 | — | — | — | — | Equity submitted; options blocked | $137.38 |

---

## Notes

- First paper run: no prior position or DB state; EOD correctly generated BUY from flat
- Morning runner submits equity + options; EOD runner generates signal + pending orders
- Reconciliation: EOD syncs IBKR positions before building orders (prevents duplicate buys)

---

## Open To-Dos

- [x] **XSP options permissions confirmed (2026-05-08)** — 198 contracts visible for Aug-21 expiry; permissions were always enabled. Root issue was `MIN_STRIKE_STEP = 0.50` requesting invalid strike 515.5; fixed to 5.0 (XSP uses $5 increments at 30% OTM). Both legs ready to execute.
- [ ] **XSP market data subscription** — IBKR returns Error 354 (not subscribed) for live XSP index price. Fallback to SPX/10 via yfinance works correctly. Optional: subscribe to CBOE index data in Market Data Subscription Manager to get live XSP prices without fallback.
