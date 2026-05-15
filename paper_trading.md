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
- Runner ready for full dual-leg execution Monday 2026-05-11

---

### 2026-05-11 — Gateway down; runner failed

**Morning Runner (9:40 ET)**
- `ConnectionRefusedError` on port 4002 — IBKR Gateway not running
- Pending BUY 713 UPRO @ $141.49 (written 2026-05-10 EOD) never submitted
- Paper account remains flat, $100K cash

---

### 2026-05-12 — Gateway down again

**Morning Runner (9:40 ET)**
- Same `ConnectionRefusedError` — gateway still not running
- Signal 3 days stale by this point

---

### 2026-05-13 — Gateway down (third consecutive failure)

**Morning Runner (9:40 ET)**
- Same `ConnectionRefusedError`

**Root cause investigation (2026-05-13/14)**

Three compounding bugs in the IBC gateway startup:

1. **Scripts not executable** — `gatewaystart.sh`, `stop.sh`, `commandsend.sh` all had `-rw-rw-r--` (no execute bit). Crontab fired but shell returned `Permission denied` immediately.

2. **Wrong IBC path** — `gatewaystart.sh` had `IBC_PATH=/opt/ibc` but IBC lives at `~/ibc`. Script would have aborted with "no execute permission for scripts in /opt/ibc/scripts" even if executable.

3. **Wrong gateway version** — `TWS_MAJOR_VRSN=1019` but installed version is `1046` (in `~/Jts/ibgateway/1046/`). IBC error: "Offline TWS/Gateway version 1019 is not installed: can't find jars folder".

**Fixes applied:**
- `chmod +x ~/ibc/*.sh ~/ibc/scripts/*.sh`
- `IBC_PATH=/opt/ibc` → `IBC_PATH=~/ibc` in `gatewaystart.sh`
- `TWS_MAJOR_VRSN=1019` → `TWS_MAJOR_VRSN=1046` in `gatewaystart.sh`
- Gateway tested manually — connected to paper account `DUQ241384` on port 4002 ✓

**Note:** IBC gateway code lives outside this repo at `~/ibc/` (IBC v3.23.0). Changes to `gatewaystart.sh` are not git-tracked. Gateway binary is at `~/Jts/ibgateway/1046/`.

---

### 2026-05-14 — Crontab updated: gateway auto-start/stop

Added two crontab entries (Mon–Fri ET):
- `9:20 AM` — `DISPLAY=:99 ~/ibc/gatewaystart.sh` (20 min before 9:40 paper runner)
- `6:00 PM` — `~/ibc/stop.sh` (25 min after 5:35 EOD paper runner)

Gateway uses Xvfb display `:99` (already running persistently). IBC auto-logs in via credentials in `~/ibc/config.ini`. Both gateway scripts are now executable.

Also fixed: **negative cash bug** in `runner_eod_v52dd12.py` — share sizing was `int(cash // upro_close)` which could order one share more than cash covers when filled at `close × 1.01`. Fixed to size at limit price: `shares = int(cash / limit_price)`. Committed 94258464.

---

## Daily Signal Summary

| Date | Signal | VIX rank | DD | MACD | Action | UPRO px |
|------|--------|----------|----|------|--------|---------|
| 2026-05-07 | UPRO | 20.5% | −0.3% | +1.36 | BUY pending | $136.02 |
| 2026-05-08 | — | — | — | — | Equity submitted; options blocked | $137.38 |
| 2026-05-11 | — | — | — | — | Gateway down — not submitted | — |
| 2026-05-12 | — | — | — | — | Gateway down — not submitted | — |
| 2026-05-13 | — | — | — | — | Gateway down — not submitted | — |

---

## Notes

- First paper run: no prior position or DB state; EOD correctly generated BUY from flat
- Morning runner submits equity + options; EOD runner generates signal + pending orders
- Reconciliation: EOD syncs IBKR positions before building orders (prevents duplicate buys)

---

---

### 2026-05-15 — Simulation DB correction

**Manual correction applied to `~/.mixa/simulation.db`:**

The simulation DB had an incorrect position caused by the negative-cash sizing bug (fixed in commit 94258464). On 2026-05-07 the EOD runner sized shares using `int(cash // upro_close)` instead of `int(cash / limit_price)`, resulting in 728 shares purchased at $138.00 fill against a $100K budget — overspending by $464 and leaving cash at −$464.

Correct sizing with the fixed formula:
- `limit_price = 138.67` (= close × 1.01, from orders table)
- `shares = int(100,000 / 138.67) = 721`
- `cost = 721 × $138.00 = $99,498`
- `cash = $502`

Correction applied:
```sql
UPDATE positions SET shares=721 WHERE ticker='UPRO';
UPDATE account_cash SET cash=502.0 WHERE id=1;
```

Paper DB (`~/.mixa/paper.db`) was unaffected — its position (718 UPRO @ $138.755) reflects actual IBKR fills and was already corrected on 2026-05-14.

---

## Open To-Dos

- [x] **XSP options permissions confirmed (2026-05-08)** — 198 contracts visible for Aug-21 expiry; permissions were always enabled. Root issue was `MIN_STRIKE_STEP = 0.50` requesting invalid strike 515.5; fixed to 5.0 (XSP uses $5 increments at 30% OTM). Both legs ready to execute.
- [ ] **XSP market data subscription** — IBKR returns Error 354 (not subscribed) for live XSP index price. Fallback to SPX/10 via yfinance works correctly. Optional: subscribe to CBOE index data in Market Data Subscription Manager to get live XSP prices without fallback.
