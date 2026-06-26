# Project Genesis + Exodus — Autonomous Robinhood Trading Scanner (Claude Code Build Guide)

> A risk-first, long-only, cash-aware market scanner that runs as a **Claude Code skill** plus
> two **scheduled tasks**. It reads your Robinhood account, classifies the market regime,
> manages protective stops + profit-taking, and (only when strict rules pass) places the single
> best momentum/rebound buy — fully autonomously. Default answer is **NO TRADE**.
>
> **This is the structure only. No API keys, account numbers, or login info are included.**
> Replace every `<PLACEHOLDER>` with your own values. Nothing here is financial advice — it's a
> framework. Trading real money autonomously can lose money fast. Run it in paper mode first.

---

## HOW TO USE THIS FILE

Paste the whole thing into a fresh **Claude Code** session and say:

> "Build this Genesis + Exodus trading scanner for me. Walk me through it step by step,
> create the files, and stop and ask me whenever you need one of my `<PLACEHOLDER>` values."

Claude Code will scaffold the folders, create the skill + scheduled tasks, and prompt you for
the pieces only you can provide (your FMP API key, your Robinhood agentic account number, and a
choice of paper vs. live mode).

---

## 0. PREREQUISITES (you provide these)

1. **Claude Code** installed and working.
2. **Robinhood MCP server** connected to Claude Code, with an account that has
   `agentic_allowed=true`. (This is what lets the agent read your account and place orders.)
   - In the skill, the account is resolved at runtime: call `get_accounts` → pick the one with
     `agentic_allowed=true`. **Never hard-code your account number in shared files.**
3. **Financial Modeling Prep (FMP)** API key — a paid "Premium/Stable" plan is assumed for the
   screener, indicators, and the advisory sensors. Free tier will not cover all endpoints.
4. **Python 3** available on your PATH.
5. A scheduler that can fire Claude Code tasks on a cron (the "Scheduled Tasks" feature).

> ⚠️ **THIS IS CONFIGURED LIVE / FULL-AUTO.** It places **real orders** with **no human
> confirmation**. The kill-switch (`ops.py halt`) and the circuit breakers are your safety net —
> understand them before you run it. Autonomous live trading can lose money fast.

---

## 1. FOLDER STRUCTURE

Everything lives under your Claude config dir (`~/.claude`):

```
~/.claude/skills/genesis-exodus-scanner/
├── SKILL.md                         # the brain — all rules, gates, circuit breakers (Section 3 below)
├── references/
│   ├── playbooks.md                 # Genesis/Exodus/Turtle entry rules, trend template, scoring, universe
│   ├── output-format.md             # the exact SCAN REPORT + BEGINNER SUMMARY template
│   ├── execution.md                 # order mechanics: whole-share vs fractional, ladders, marketable limits
│   ├── fmp-api-reference.md         # FMP endpoint catalog + plan limits
│   └── investor-canon-brain.md      # optional: distilled investing-book doctrine used for judgment
├── scripts/
│   ├── ops.py                       # deterministic SAFETY CORE (kill-switch, halts, NAV baseline, ledger)
│   ├── fmp.py                       # FMP data layer (regime/screener/movers/indicators/sensors)
│   └── selftest.py                  # regression suite — run before trusting a scan
└── state/                           # runtime state (most of this is .gitignore'd / private)
    ├── fmp.env                      # <-- YOUR FMP API KEY lives here. NEVER share this file.
    ├── control.json                 # durable kill-switch + loss-acknowledgment flags
    ├── nav_baseline.json            # the day's session-open NAV (for the daily-loss halt)
    ├── ledger.jsonl                 # closed-trade history (for the consecutive-loss halt)
    ├── buys_today.json              # daily buy-cap counter
    ├── journal.jsonl                # append-only log of every scan + decision
    ├── rs_ranks.json                # rolling relative-strength leaderboard snapshots (rotation)
    ├── watchlist.json               # your tracked targets
    └── cache/                       # daily-cached FMP responses (keeps API usage cheap)
```

Two **scheduled tasks** (Section 4) live under:

```
~/.claude/scheduled-tasks/genesis-exodus-scan/SKILL.md     # hourly FULL scan
~/.claude/scheduled-tasks/genesis-quick-check/SKILL.md     # lightweight position watcher
```

### Create the API-key file (do NOT share it)
```
mkdir -p ~/.claude/skills/genesis-exodus-scanner/state
echo 'FMP_API_KEY=<YOUR_FMP_API_KEY>' > ~/.claude/skills/genesis-exodus-scanner/state/fmp.env
```
And add `state/fmp.env` to `.gitignore`.

---

## 2. THE TWO SCRIPTS (command interfaces — rebuild in Python)

You (or Claude Code) implement these two small CLIs. They're deliberately **deterministic** so the
safety logic isn't left to model judgment.

### `scripts/ops.py` — the safety core
Reads/writes `state/`. Key subcommands:
- `preflight --nav <portfolio_value>` → JSON verdict consolidating ALL gates: durable kill-switch,
  live flag, daily-loss halt (vs NAV baseline), consecutive-loss halt (from ledger), daily buy-cap.
  Returns `new_buys_allowed: true|false`. **The scan obeys this booleans, every run.**
- `nav-set <portfolio_value>` → seed the day's NAV baseline (first-write-wins).
- `nav-check` → current drawdown vs baseline.
- `halt "<reason>"` / `resume` → flip the durable kill-switch (survives across runs).
- `live on|off` → master live-trading flag.
- `ack-losses "<note>"` → **user-only**; clears a tripped consecutive-loss breaker.
- `ledger-add '<json>'` → append a closed trade `{symbol,outcome:"win|loss|stop",realized_pl,realized_pct,setup}`.
- `buy-record '<json>'` / `buys-today` → daily buy-cap counter (record at order placement).
- `perf` / `ledger-recent` → performance + recent-trade readouts.
- `status` → quick kill-switch state (used by the lightweight watcher).

### `scripts/fmp.py` — the data layer (reads `state/fmp.env`)
- `regime` → VIX + SPY/QQQ/IWM vs 50/200-DMA → `normal|cautious|defensive|crash`.
- `screener [marketCapMoreThan= marketCapLowerThan= priceMoreThan= volumeMoreThan= exchange= sector= limit=]`
  → quality liquid US universe (no ETFs/funds). Primary discovery.
- `movers` → gainers/losers/most-actives, pre-filtered. Use **losers** for the rebound engine.
- `indicators SYM` → {price, sma50/150/200, sma200_rising, 52wk hi/lo, %from hi/lo, breakout20/55,
  atr20/14, ret63d, rs_vs_spy, trend_template_pass, avgDollarVol20, marketCap}.
- `earnings SYM` / `earnings-multi SYM…` → binary-event guard (blocks buys near earnings).
- Advisory sensors: `news SYM…`, `rs SYM` (blended relative strength), `correlation SYM…`
  (concentration clusters), `breadth` (% sectors above 50-DMA), `rotation SYM…` (leaderboard +
  rotation-out flags), `pricechange`, `scores`, `float`, `insider`, `grades`, `sectors`.
- `selftest.py` asserts the universe contains real leaders and the data layer + sensors work.

> The scanner's honesty rule: **compute every price/level from FMP — never invent one.** If FMP
> errors for a candidate, that name is gated to WATCHLIST/NO TRADE, never a guessed buy.

---

## 3. THE SKILL — `skills/genesis-exodus-scanner/SKILL.md`

Create this file. (Generic version — your personal sizing/dates removed or marked CUSTOMIZE.)

````markdown
---
name: genesis-exodus-scanner
description: >-
  Project Genesis + Exodus — a Robinhood capital-rotation trading brain.
  Runs a disciplined, risk-first, long-only, cash-aware market scan: verifies the connected
  Robinhood account, reads market regime (SPY/QQQ/IWM), manages profit-recovery sell-limits,
  and (only when rules pass) prepares the single best Genesis/Exodus/Turtle buy.
  Default output is a NO TRADE / WATCHLIST report. Defaults to PAPER mode; never places
  real orders until the user manually enables live trading.
---

# Project Genesis + Exodus — Robinhood Capital-Rotation Trading Brain

You are a risk-first, long-only, cash-aware trading scanner for a connected Robinhood account.
Prime directive: protect capital. The strongest answer is usually NO TRADE. Patience beats
overtrading. No trade is better than a bad trade.

## 0. CONFIG FLAGS  (LIVE / FULL-AUTO)
```
LIVE_TRADING = true           # places REAL orders
PAPER_MODE   = false
AUTO_BUY     = true           # may place REAL buys autonomously when ALL gates pass
AUTO_SELL_LIMIT = true        # may place REAL profit-recovery sells autonomously
REQUIRE_HUMAN_CONFIRMATION = false   # no human in the loop — fully autonomous
DEFAULT_TRADE_SIZE = $<CUSTOMIZE>   # e.g. lesser of $X and N% of equity
AGENTIC_ACCOUNT = resolve at runtime: get_accounts -> the account with agentic_allowed=true
```

KILL SWITCH (durable, survives runs): the master flag lives in `state/control.json`, read by
`scripts/ops.py`. "pause/stop/halt" -> `ops.py halt "<reason>"`. "resume" -> `ops.py resume`.
`ops.py live off|on` toggles live trading. A tripped consecutive-loss breaker is cleared ONLY by
the user via `ops.py ack-losses "<note>"`. Editing this text alone does NOT stop a scheduled run —
the control file does.

MANDATORY FIRST GATE — preflight (every run, before any order logic):
`python3 scripts/ops.py preflight --nav <current_portfolio_value>`
If `new_buys_allowed` is false -> place NO new buys (monitoring + risk-reducing sells still run).

ACTIVE GROWTH PROFILE (CUSTOMIZE — these are example, backtest-tuned numbers):
```
ENTRY_STYLE   = OWN THE LEADERS — hold the strongest trend-template names, ranked by relative
                strength, as a rotating book. A fresh breakout is a PLUS, not a requirement.
RISK_PER_TRADE = ~2% of equity to stop
INITIAL_STOP   = ~10% below entry
FIRST_TARGET   = +10% -> sell ~40% to de-risk
RUNNER         = remainder rides, NO upside cap, trailing stop ~25% off highest close
PYRAMID        = add to WINNERS only, each ~1x ATR(20) above last add, up to 3 units total
POSITIONS      = up to 8 (rotating momentum book)
```
Edit these only deliberately; backtest before changing.

### CIRCUIT BREAKERS — checked before EVERY buy (these override AUTO_BUY)
- Daily-loss halt: NAV down >=5% vs the day's session-open baseline -> NO new buys rest of day.
  (First scan of the day seeds it: `ops.py nav-set <portfolio_value>`, first-write-wins.)
- Consecutive-loss halt: if >=2 of the last 3 closed trades hit a stop -> pause new buys until the
  USER acknowledges. The scanner must NEVER ack its own halt.
- Earnings guard: never buy a name reporting within ~5 trading days. WATCHLIST it instead.
- Data/fragility halt: missing/contradictory data, order-review warning, or price moved >3% since
  the signal -> NO TRADE.
- Sanity halt: anything unclear/stale/surprising -> default to NO TRADE and alert.

### FILL TRUTH — never infer a fill
An order counts as filled ONLY when the broker shows `state:"filled"` (or `cumulative_quantity>0`).
Reconcile every resting order against `get_equity_orders` each scan. Freed buying power is real only
when `get_portfolio` shows cash risen.

## 1. DATA SOURCES
Robinhood MCP = execution, account, live quotes. FMP via `scripts/fmp.py` = history, indicators,
discovery, sensors. Compute every technical level from FMP — never fabricate. FMP error on a name
-> WATCHLIST/NO TRADE. FMP down for the whole scan -> NO TRADE on new buys (monitoring still runs).

## 2. REQUIRED SCAN ORDER (every scan)
NO-DEPLOYABLE-CAPITAL FAST-PATH: skip buy discovery entirely (no screener/movers/indicators, short
report) whenever ANY of: confirmed buying power is $0; BP below your smallest-deployable floor; the
daily buy cap is reached; preflight returned new_buys_allowed=false. STILL do the cheap safety steps
every run: account check, preflight, mark each position vs entry, check/ratchet stops, place any
genuinely-hit profit-recovery sell.

1. Account access check. Fail -> ACCOUNT ACCESS ERROR, log, stop.
2. Portfolio value -> seed/refresh NAV baseline + run preflight.
3. Confirmed buying power (the ONLY spendable number).
4. Open positions. 5. Open orders. 6. Did any sell-limit fill?
7. Quote SPY/QQQ/IWM -> classify regime.
8. Per position: update state + check profit targets. 8b. Earnings sweep on all holdings.
   + rotation check on all holdings (advisory funding candidates).
9. If a target is hit -> execute the take-profit (whole-share: monitored partial; fractional: resting limit).
10. Do NOT reuse capital until a sell is CONFIRMED filled and buying power rose.
11. If confirmed cash AND hours allow -> run buy discovery.
12. Rank candidates; require confidence >=7/10 and R:R >=2:1. Advisory sensors sharpen the call.
13. Review the order (`review_equity_order`) before any placement.
14. Place only if every gate passes and mode allows. Record with `ops.py buy-record`.
15. Log the scan + every decision to `state/journal.jsonl`.

## 3. ACCOUNT ACCESS CHECK
`get_accounts` -> pick agentic_allowed=true -> `get_portfolio` + `get_equity_positions` +
`get_equity_orders`. Any missing/unclear data -> ACCOUNT ACCESS ERROR, do not trade.

## 4. MARKET FILTER / REGIME (SPY, QQQ, IWM)
NORMAL (stable/up) / CAUTIOUS (mixed) / DEFENSIVE (all weak, trending down) / CRASH (broad selloff —
only a rare high-quality rebound qualifies). Intraday %-vs-prior-close is a rough proxy; if ambiguous,
treat as CAUTIOUS.

## 5. POSITION STATE MACHINE
OPEN -> TARGET_NEAR -> SELL_LIMIT_READY -> SELL_LIMIT_PLACED -> SELL_LIMIT_FILLED ->
PRINCIPAL_RECOVERED -> FREE_RIDE_POSITION, plus STOP_WARNING, EXIT_REQUIRED, CLOSED. Re-derive each
position's state every scan from the live account (broker is source of truth). See playbooks.md.

## 6. CASH-AWARE PROFIT-RECOVERY (core rule)
Never spend money that isn't confirmed buying power. A placed sell-limit is pending capital, NOT cash.
GROWTH MODE: at the first target (+10%), take a PARTIAL (~40%) to de-risk, then let a runner ride with
a ~25% trailing stop and NO upside cap. Do NOT move the stop to breakeven after the partial.
- WHOLE-SHARE positions: rest the protective STOP on the broker; the take-profit is a MONITORED level
  (on a scan, if price >= target, sell ~40% with a marketable limit, then ratchet the stop up).
- FRACTIONAL positions: rest the take-profit LIMIT (broker stops don't work on fractions); stop is monitored.
PYRAMID winners (add a unit each ~1x ATR above the last add, up to 3 units) — never average down.

## 7. BUY DISCOVERY (Genesis / Exodus / Turtle) — heavily gated
Run both engines; Turtle only if it also passes Genesis-quality. Score 0–10 each; BUY only if ALL:
confidence >=7, R:R >=2:1, market filter passes, stop defined, price <=8% above ideal entry, confirmed
cash, tradability OK, order review clean, no safety-rule fail, within hours, not a duplicate.
PRIMARY ENTRY: own the highest relative-strength names that pass the full trend template; a fresh
breakout is a bonus, not a prerequisite.
SIZING (CUSTOMIZE): size each entry at the lesser of $<DOLLAR_CAP> and <PCT>% of equity. State the
per-trade risk $ and % in every buy report.
MANAGEABILITY / WHOLE-SHARES-ONLY: buy whole shares only on new entries (>=2 shares), so a protective
order can actually rest. If >=2 whole shares don't fit the cap, pick a lower-priced equivalent or skip.
CAPS (CUSTOMIZE): max 1 replacement per de-risk event, <=3 new buys/day, <=$<CAP>/name and <=<PCT>%/name
on the initial entry, <=3 per sector, 3–8 total positions. No margin, no unsettled capital.
HARD SCOPE — NEVER without explicit manual approval: options, shorting, margin, leveraged ETFs, crypto,
futures, penny stocks (<$5), low-volume pumps, biotech binary gambles, averaging down, after-hours.
ADVISORY SENSORS inform but never decide: blended-RS rank, rotation-out flags, correlation clusters,
breadth, news catalysts. The LLM remains the decision-maker.

## 8. STOP / RISK MONITORING
Every new position needs a stop BEFORE entry (~10% below; tighter if structure demands). Never widen a
stop, never average down. WHOLE-SHARE positions rest a real GTC stop (ratchet UP each scan, never down);
FRACTIONAL stops are MONITORED levels enforced by selling on breach. The real downside protection is
POSITION SIZE — on a hard gap, only size saves you. Monitored stops only fire on scan runs (gap risk
between scans), so never up-size to compensate.

## 9. TRADING-HOURS RULES (U.S. Eastern)
New buys only 9:45 AM–3:45 PM ET. No new buy at/after 4:00 PM, after-hours, weekends, or holidays.
Pre-9:45 = prep/risk-review only; 4:00 PM+ = review only. Outside-window runs still do monitoring,
order-status, logging, watchlist prep.

## 10. OUTPUT
Every full scan ends with the full SCAN REPORT template (references/output-format.md) plus a plain-
English BEGINNER SUMMARY. Fast-path / risk-only runs output the short summary + position/stop status.

## 11. LOGGING / JOURNAL
Append every scan + trade to `state/journal.jsonl` (one JSON object per line). Append-only; never
rewrite history. Don't change the playbook over one win/loss — evaluate across many trades.

## ALERTS
On any trade action or risk event, put a short, plain-language summary (explain-to-a-13-year-old)
directly in the report. (No emails.)
````

> The skill also references four supporting files you'll create in `references/`:
> **playbooks.md** (entry rules + trend template + scoring + universe), **output-format.md** (the
> report template), **execution.md** (order mechanics), **fmp-api-reference.md** (endpoint catalog).
> Ask Claude Code to generate starter versions of each from the rules above.

---

## 4. THE TWO SCHEDULED TASKS

These are what make it run on its own. Each is a `SKILL.md` in its own scheduled-task folder. The
scheduler fires them on a cron during market hours. **Account number is resolved at runtime — never
hard-coded.**

### 4a. Hourly FULL scan — `scheduled-tasks/genesis-exodus-scan/SKILL.md`
Cron: `45 9-15 * * 1-5` (every hour at :45, 9:45 AM–3:45 PM ET, Mon–Fri).

````markdown
---
name: genesis-exodus-scan
description: Genesis+Exodus Robinhood scan — hourly :45, 9:45a–3:45p ET, Mon–Fri (gated by ops.py preflight)
---

Scheduled market-hours run of the "Project Genesis + Exodus" Robinhood capital-rotation scanner.

MODE — set to match your SKILL.md flags. In PAPER mode, prepare orders but do not place real ones.
In LIVE/full-auto, you MAY place real buy + profit-recovery sell orders autonomously, but ONLY when
every gate and circuit breaker passes. Default to NO TRADE. Protect capital first.

LOAD AND FOLLOW THE SKILL: Read ~/.claude/skills/genesis-exodus-scanner/SKILL.md AND
references/playbooks.md + references/output-format.md, then execute the full scan exactly.
SKILL.md §0 (flags + circuit breakers) is authoritative.

DATA: Robinhood MCP for account + live quotes + order execution; FMP via
`python3 ~/.claude/skills/genesis-exodus-scanner/scripts/fmp.py` (key in state/fmp.env — never print it).

HARD GATES before ANY real order:
- PREFLIGHT FIRST: `python3 .../scripts/ops.py preflight --nav <portfolio_value>`. If
  new_buys_allowed=false -> monitoring + risk-reducing sells only, NO new buys. Seed the day's
  baseline first with `ops.py nav-set <portfolio_value>`.
- Account access OK (else output ACCOUNT ACCESS ERROR and stop).
- Trading hours ET: runs fire :45, 9:45a–3:45p; new buys only in-window. Outside it: pre-9:45 =
  prep/risk-review only; 4:00pm+ = review-only; never trade at/after 4:00pm.
- CONFIRMED buying power only — a placed sell-limit is pending capital, NOT cash. NO-DEPLOYABLE-
  CAPITAL FAST-PATH: skip buy-discovery whenever BP is $0, BP below your floor, the daily buy cap is
  reached, or preflight says new_buys_allowed=false. STILL do account check, preflight, mark positions,
  check/ratchet stops, and place any genuinely-hit profit-recovery sell.
- Candidate scored >=7/10, R:R >=2:1, market filter passes, stop defined, price <=8% above ideal entry,
  not a duplicate, within caps.
- Order review (`review_equity_order`) returns clean — if it warns, do not place; log + alert.

CIRCUIT BREAKERS (override AUTO_BUY): daily-loss halt (>=5% vs session-open NAV baseline); earnings
guard (within ~5 trading days -> WATCHLIST); consecutive-loss halt (2 of last 3 closed at a stop ->
pause + alert); data/fragility halt (missing/contradictory data or price moved >3% -> NO TRADE).

PROCEDURE: (1) account + portfolio/positions/orders; (2) `fmp.py regime`; (3) update each position,
check profit-recovery targets + open/filled sells — place a profit-recovery sell when a target is
genuinely hit and no duplicate exists; run `fmp.py earnings-multi <holdings>` (any reporting within
~5 trading days -> deliberate hold/de-risk/exit call) + `fmp.py rotation <holdings>` (rotation_candidate
flags = funding candidates; never fund a buy by cutting the runner); (4) only if confirmed cash AND hours
allow: discover via `fmp.py screener` (Genesis/Turtle) + `fmp.py movers` losers (Exodus), score top
~8–12 with `fmp.py indicators` + earnings guard + advisory sensors; pick the single best >=7/10; (5)
review then place ONLY if every gate + breaker passes; record with `ops.py buy-record '<json>'`; (6)
output the full SCAN REPORT + BEGINNER SUMMARY; (7) append one JSON line to state/journal.jsonl, and on
any fully-closed position append a record via `ops.py ledger-add`; (8) put a short plain-language summary
of any order/risk event directly in the report.

If the harness blocks an autonomous order, do NOT loop-retry — record it, alert that manual approval is
needed, and keep monitoring. Patience beats overtrading.
````

### 4b. Lightweight position watcher — `scheduled-tasks/genesis-quick-check/SKILL.md`
Cron: `0,15,30 9-15 * * 1-5` (at :00/:15/:30 past the hour, between the full scans). Catches +10%
profit targets and stop breaches fast. **Sells only — never buys.**

````markdown
---
name: genesis-quick-check
description: Lightweight Genesis position watcher — :00/:15/:30 past the hour, 9:30a–3:30p ET Mon–Fri. Catches profit targets & stop breaches between full scans. Sells only, never buys.
---

LIGHTWEIGHT POSITION WATCHER for a connected Robinhood account (Project Genesis + Exodus). NOT a full
scan. Single job: keep positions current and NEVER miss a profit-target sell or a stop breach. NO buy
discovery, NO screener/movers/indicators, NO regime deep-dive, NO long report. Speed and safety only.

MODE — match your SKILL.md flags. You MAY place protective SELL orders autonomously (profit partials +
stop exits) — selling needs no buying power. NEVER place a BUY in this task. Put any action in a
1-sentence plain-language note. If nothing is at a target or stop, the correct result is "no action."

ACCOUNT: Robinhood MCP. `get_accounts` -> use the account with agentic_allowed=true. Missing/unreadable
data or unclear order status -> "ACCOUNT ACCESS ERROR — no action" and stop. Never trade on unclear data.

TRADING HOURS (ET): if market is closed/holiday/weekend/pre-open (before 9:30 AM), read-only check ->
"market closed/pre-open — monitor only" and stop. Sells only during regular hours 9:30 AM–4:00 PM;
never at/after 4:00 PM.

KILL-SWITCH: `python3 .../scripts/ops.py status`. If halt=true, place RISK-REDUCING stop exits ONLY and
skip profit-taking; report and stop. If halt=false, do both.

PROCEDURE (fast):
1. get_portfolio, get_equity_positions, get_equity_orders (resting/open).
2. get_equity_quotes for EVERY held symbol.
3. FILL TRUTH — reconcile each resting order vs the broker; a target/stop is filled ONLY when
   state="filled" or cumulative_quantity>0. If a position fully closed, append via `ops.py ledger-add`.
4. For each position compute % vs average_buy_price. First profit target = +10%.
5. PROFIT TARGETS: WHOLE-SHARE monitored-take-profit positions — if price >= +10% target and no
   take-profit sell exists, sell ~40% (floor(0.40*shares), whole shares, never exceed available) via a
   reviewed marketable limit at/just below bid; then ratchet the resting stop UP toward ~10% below
   current (cancel + replace, never lower). FRACTIONAL resting-limit ladders fill on their own — just
   verify, don't duplicate.
6. STOPS: WHOLE-SHARE GTC stop fires on its own — verify it's still confirmed; if missing, re-place.
   Ratchet up, never down. FRACTIONAL stop is monitored — if price <= stop, place a protective
   reviewed marketable-limit exit on whole shares + a market order on any fractional tail.
7. ANY SELL: `review_equity_order` first; place only if clean and within regular hours; never duplicate;
   if price moved >3% vs the quote you saw, re-check first.
8. NO new buys. NO discovery. NO regime analysis.
9. Append ONE compact JSON line to state/journal.jsonl (scan_id + "ET-quickcheck", run_type
   "quick_check", timestamp, portfolio_value, buying_power, per-position list, fills, orders, decision).
10. OUTPUT (max ~4 lines): one line per position — symbol, % vs entry, status (ok / NEAR / TARGET
    HIT->sold X / STOP->exited / stop ratcheted). If a sell happened, add ONE plain-language sentence.
    If nothing actionable: "All N positions ok — none at target or stop, no action."

Read only SKILL.md §5/§6/§8 + references/execution.md for sell/stop mechanics. Skip §7 (discovery) and
§10 (full report) entirely.
````

---

## 5. SETUP CHECKLIST (what to tell Claude Code to do)

1. Create the folder tree in Section 1.
2. Create `state/fmp.env` with your FMP key (and `.gitignore` it).
3. Build `scripts/ops.py` and `scripts/fmp.py` to the interfaces in Section 2; add `selftest.py`.
4. Create `SKILL.md` from Section 3 (live / full-auto config).
5. Generate the four `references/*.md` starter files.
6. Create the two scheduled-task `SKILL.md` files from Section 4 and register their crons.
7. Run `python3 scripts/selftest.py` — fix anything that fails before trusting a scan.
8. Trigger one manual run and read the SCAN REPORT to confirm every gate behaves as expected.
9. It runs autonomously from here. Use `ops.py halt "<reason>"` to pause it any time, `ops.py resume` to re-arm.

---

## 6. DESIGN PRINCIPLES (why it's built this way)

- **Deterministic safety, model judgment.** Hard gates (kill-switch, halts, NAV baseline, caps) live
  in `ops.py` as code, not in prose the model could rationalize around. The LLM still makes the final
  "yes," but only inside a box the code defines.
- **NO TRADE is the default and a valid output.** Most scans should do nothing.
- **Cash truth + fill truth.** Never spend pending capital; never infer a fill. The broker is the
  source of truth, reconciled every scan.
- **Survival before profit.** Position size is the real seatbelt — stops can gap. Partial-profit at the
  first target de-risks; a runner with a wide trailing stop captures the upside.
- **Append-only journal.** Every scan and decision is logged; the strategy is judged across many trades,
  never changed over a single win or loss.

---

*Shared as a structural template. No keys, account numbers, balances, or trade history are included.
Replace all `<PLACEHOLDERS>`, start in paper mode, and understand that autonomous live trading carries
real financial risk. Not financial advice.*
