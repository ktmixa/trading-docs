# Paper Trading Launch Plan — MixA V52.DD.OTM

**Status:** Ready to execute  
**Target:** Switch both runners from `--mode simulation` to `--mode paper`

**Decisions made:**
- Shared IBKR account (`mixairaalgo`) with a distinct client ID (`IBKR_CLIENT_ID=3`)
- Penny uses client 2 on port 4002; mixa uses client 3 on same gateway
- Starting NAV: full IBKR paper balance (~$1M); position sizing scales to that NAV
- Positions bug resolved: EOD runner now calls `_sync_ibkr_positions` before building orders — reads actual IBKR holdings so it cannot generate duplicate orders after a fill

---

## 1. Remaining setup (one-time)

---

## 2. ibgw: no new gateway needed

Mixa connects to penny's existing `ibgw` gateway instance (port 4002) with a different client ID. No new `start.sh` registration or IBC config required.

### 2a. Fix stop.sh bug (do this regardless)

`stop.sh` references `$PORT` in its wait loop but never sets it. Currently harmless for one instance but would silently skip the port check. Fix:

```bash
# Add after the INSTANCE/LOG_FILE lines at top of stop.sh:
case "$INSTANCE" in
    penny) PORT=4002 ;;
    *)     PORT="${IBKR_PORT:-4002}" ;;
esac
```

### 2b. Confirm penny gateway is included in crontab start/stop

Mixa's morning runner at 9:35 needs the gateway already up. Check penny's crontab — if penny starts the gateway at 9:20, mixa benefits automatically. If penny only starts it on demand, add a start line to mixa's crontab (or coordinate).

---

## 3. mixa/.env additions

```bash
# Append to /home/brewadmin/projects/trading/mixa/.env:
IBKR_PORT=4002        # penny's gateway port
IBKR_PAPER=True
IBKR_CLIENT_ID=3      # penny uses client 2; mixa uses client 3
IBKR_HOST=127.0.0.1
```

`IBKRExecutor.__init__` reads all four; defaults are 7497 / True / 2 / 127.0.0.1.

---

## 4. Day-1 bootstrap (paper DB cold start)

On the first paper-mode run there is no position and no prior signal history in `paper.db`. The EOD runner will:

1. Compute today's V52.DD12 signal fresh from market data (correct — no dependency on prior DB state for signal computation)
2. Read `positions` → empty → compute desired = UPRO if signal is on
3. Write a pending BUY order

The morning runner will then submit the buy. This is correct behavior. **Just make sure there is no stale pending order in `paper.db` from a prior test run.**

**Day-1 checklist:**
```bash
# Confirm paper.db is clean (or does not exist):
ls ~/.mixa/paper.db  # should not exist, or:
sqlite3 ~/.mixa/paper.db "SELECT * FROM pending_orders;"  # should be empty

# Confirm gateway responds (penny gateway must be running):
python3 -c "from ib_insync import IB; ib=IB(); ib.connect('127.0.0.1', 4002, clientId=3); print(ib.accountValues()[:3]); ib.disconnect()"

# Dry-run both runners against paper gateway:
venv/bin/python3 live/runner_eod_v52dd12.py --mode paper --dry-run
venv/bin/python3 live/runner_morning.py --mode paper --dry-run
```

---

## 5. Crontab changes

Replace the current crontab with:

```
TZ=America/New_York

# Morning runner: equity + options (penny gateway must already be running)
35 9  * * 1-5 /home/brewadmin/projects/trading/mixa/venv/bin/python3 /home/brewadmin/projects/trading/mixa/live/runner_morning.py --mode paper >> /home/brewadmin/projects/trading/mixa/logs/morning.log 2>&1

# EOD runner: signal + IBKR position sync + pending orders
30 17 * * 1-5 /home/brewadmin/projects/trading/mixa/venv/bin/python3 /home/brewadmin/projects/trading/mixa/live/runner_eod_v52dd12.py --mode paper >> /home/brewadmin/projects/trading/mixa/logs/eod.log 2>&1
```

No gateway start/stop lines needed — penny's gateway (port 4002) is shared. Coordinate with penny's crontab to ensure the gateway is up before 9:35.

---

## 6. Monitoring the first week

**Log locations:**
```
logs/gateway.log   — gateway start/stop events and auth
logs/morning.log   — equity fill + options order submission
logs/eod.log       — signal, pending order generation, fill simulation
```

**Daily checks:**
- EOD log: confirm signal computed, pending order written with correct limit price
- Morning log: confirm `Connected to IBKR PAPER`, order submitted, fill audit line
- IBKR paper account (TWS or portal): confirm position matches `paper.db`
- `sqlite3 ~/.mixa/paper.db "SELECT * FROM positions;"` — should show UPRO shares after first fill
- `sqlite3 ~/.mixa/paper.db "SELECT * FROM put_position;"` — shows XSP put after first ENTER

**Red flags to act on immediately:**
- Gateway fails to start → morning runner will error; check `gateway.log`
- Fill price missing from morning log → order submitted but not yet filled (DAY limit may expire unfilled)
- `positions` table empty after confirmed fill → positions bug has resurfaced
- Reconciliation log shows MISMATCH → `_reconcile_position` handled it, but review the discrepancy

---

## 7. Ordered launch sequence

1. [x] Decide shared account (clientId=3) + $1M default NAV
2. [x] Add `_sync_ibkr_positions` to EOD runner — positions now always reconcile from IBKR before order generation
3. [ ] Fix `stop.sh` PORT bug
4. [ ] Add IBKR env vars to `mixa/.env` (`IBKR_PORT=4002`, `IBKR_PAPER=True`, `IBKR_CLIENT_ID=3`)
5. [ ] Confirm penny gateway is up and accepting connections, then: `python3 -c "from ib_insync import IB; ib=IB(); ib.connect('127.0.0.1', 4002, clientId=3); print(ib.accountValues()[:3]); ib.disconnect()"`
6. [ ] Paper dry-run: `venv/bin/python3 live/runner_eod_v52dd12.py --mode paper --dry-run`
7. [ ] Paper dry-run: `venv/bin/python3 live/runner_morning.py --mode paper --dry-run`
8. [ ] Confirm `~/.mixa/paper.db` is clean (no stale pending orders)
9. [ ] Update crontab: change both `--mode simulation` → `--mode paper`
10. [ ] Watch day-1 morning log; verify IBKR sync log line and equity order submission

---

## 8. Promote to live (future)

When paper trading has run correctly for 2–4 weeks:
- Change `IBKR_PAPER=False` in `mixa/.env`
- Update `IBKR_PORT` to live gateway port (4001 for IB Gateway live)
- Update `mixa.env` `IBKR_MODE=live`
- Update `mixa.ini` `TradingMode=live`
- Change crontab `--mode paper` → `--mode live`
- Consider position sizing: paper runs at full IBKR paper NAV (~$1M); confirm live NAV and that share calculation is correct
