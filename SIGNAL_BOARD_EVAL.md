# AU Signal Board — Full Evaluation Report
## Date: April 1, 2026

---

## EXECUTIVE SUMMARY

- **Overall grade: 4/10**
- **Critical issues: 9**
- **Pages functionally working: 2/4** (index, architecture partially)
- **Investor-ready: NO**
- **Data pipeline: Active** — dashboard_data.json updated 16:14 UTC today
- **Primary verdict:** The infrastructure exists and the data flows, but key display logic is broken, ledger data is wrong, and no page answers the core question: "What should I trade right now?"

---

## PAGE-BY-PAGE AUDIT

---

### Signal Board (index.html)

**Data source:** `data/dashboard_data.json` via `fetch()` + 60s auto-refresh  
**Libraries:** IBM Plex Mono + Chakra Petch (Google Fonts)

#### Working ✅
- Fleet Grid (11 assets) renders correctly from `fleet_summary` — Eagle, Nano, CVD, RSI, EMA%, R:R, Trend all displaying
- SYMPHONY OVERRIDE banner fires correctly (reads `symphony.state`)
- Macro section (F&G bar, Weatherman, Bias) works
- Catalyst Timeline renders 5 upcoming events with correct color-coding and countdowns
- Header auto-updates data age every 10s
- Navigation links to all 4 pages
- Responsive at 600px breakpoint

#### Broken ❌
1. **Portfolio Value shows $0** — `balance.portfolio_value` is `0.0` in the JSON. The agent writes `available: 1681.78` but `portfolio_value: 0.0`. Snapshot bar will always show `$0` until the agent is fixed.

2. **Edge Scanner is garbage** — All 10 `top_edges` entries in the JSON are DOGE UNDER contracts (`BUY_NO` signal) at strikes `$0.005–$0.05` with market price `99¢`. These are deep ITM NO plays with zero actionable edge:
   - `mkt_price: 99.0` rendered as `99¢` in the Mkt column — correctly shows the problem
   - `direction: UNDER` → `isBull = false` → shows as `NO` tag (red) — display is technically accurate but the contracts should never have passed filtering at source
   - `tier: PROPHECY` shown in grey (`.tag-m`) — PROPHECY doesn't trigger yellow/green highlighting, it falls to the default `tag-m` color because the tier check only handles "RED" and "GOLD" substrings
   - **The display code has NO price filter** — the `renderTopEdges()` function only filters `edge_pct >= 8`. A 99¢ deep-ITM contract passes this filter trivially
   - **Section will always be visible** because 10 garbage edges exist in the JSON

3. **Positions show "No open positions"** — `positions: []` in JSON. Either the agent isn't writing position data, or there are genuinely no open positions. Either way the section is empty and provides no value.

#### Missing 🔇
4. **No Claude/Gemini Daily Pick section** — The operator's daily trade recommendation used to appear here. Confirmed gone.
5. **No Weekly Picks / Suggestions section** — No week's top setups, no avoid list, no strategic context.
6. **No "TRADE NOW" verdict card** — The page shows data but makes no synthesis. A trader cannot act on this page in 30 seconds.

**UX Grade: 4/10**

> The fleet grid is actually useful — RSI, Eagle phase, Nano, CVD in one table is good signal board UX. But the broken $0 portfolio, garbage edge scanner taking up prime real estate, and missing daily pick make this page frustrating rather than helpful. F&G=8 (EXTREME FEAR) showing next to STRONG LONG macro bias with no explanation creates cognitive dissonance. A trader reading this cold wouldn't know what to do.

---

### Ping Pong Dashboard (dashboard.html)

**Data source:** `data/dashboard_data.json` via `fetch()` + implicit refresh on Refresh button  
**Libraries:** lightweight-charts@4.1.3, system UI fonts (not monospace)

#### Working ✅
- Charts render via lightweight-charts@4.1.3 — correct implementation
- EMA 200 price line draws (from `btc.ema200` / `eth.ema200`)
- Floor and ceiling level lines draw (f1h, f4h, c1h, c4h from `levels`)
- Timeframe buttons (4H/12H/24H/48H) correctly apply visible range via `timeScale().setVisibleRange()`
- ResizeObserver handles chart width responsively
- Cycle Tracker table structure is correct
- Fleet Signals badges section renders all BTC/ETH indicators
- Short Side Watch panel renders (currently INACTIVE since BTC/ETH are ACCUMULATION)
- Catalyst timeline renders
- Ping Pong Rules footer is concise and useful

#### Broken ❌
1. **Charts have almost no data** — `price_history.BTC.series` has 24 data points spanning 04:10–15:44 UTC today (~12 hours). `buildCandles()` uses 30-min buckets, producing only ~23 candles. At the default 48H view, the chart will show 23 tiny candles bunched in the right half with empty space on the left. The BTC range is only $68,011–$69,104 — essentially flat. The chart is technically rendering but communicates nothing useful.

2. **Portfolio value shows $0** — same JSON bug as Signal Board. `bal.portfolio_value = 0.0` → nav shows "Portfolio: $0.00". The Cash field correctly shows $1,681.78.

3. **No position overlays** — `positions: []` means no amber position strike lines, no sell order lines. The chart shows EMA + floor/ceiling only. The "Position" entry in the chart legend is misleading since nothing will ever render there while positions are empty.

4. **Cycle Tracker shows "No open positions"** — entire cycle table is empty. The page's core feature is dead when no positions exist.

5. **Draggable price lines non-functional** — Code at line 574 has a comment: `"Note: draggable price line drag detection is via subscribeCustomSeriesPriceLine which isn't available."` The `draggable: true` option is set but the drag callback is not wired up. The scenario panel (`#scenario-panel`) never appears. This is a dead feature.

6. **No verdict / synthesis** — The dashboard shows indicators (Eagle, Nano, CVD, Symphony) but never answers "BUY / WAIT / AVOID." The `renderFleet()` function lists badges but draws no conclusion. A trader at 2 AM looking at this gets data noise, not a decision.

7. **`renderChartSection()` called on first load, destroys DOM on re-render** — The code correctly avoids re-calling `renderChartSection()` after first load (comment: "it would destroy chart-btc/chart-eth DOM nodes and orphan instances"). But if `renderStaticSections()` is called, it re-renders `fleet-section` and `cycle-section`. This is correct. However, `renderChartSection()` is only called once, so the price labels (`btc-price-label`, `eth-price-label`) update correctly on subsequent fetches.

#### Missing 🔇
8. **No "ENTER / HOLD / EXIT" verdict card** — the single most useful thing for a trader
9. **No position-aware proximity alerts** ("BTC is $200 from your $69,800 strike")
10. **No macro event impact assessment** — catalysts show countdowns but not "how does NFP affect my open BTC position?"
11. **Historical chart backfill** — 12 hours of price history is not enough context for the Ping Pong strategy

**UX Grade: 3/10**

> The structure is there — chart + cycle tracker + signals is the right layout. But when all three primary display elements are empty or near-empty (no candle data, no positions, no verdict), the dashboard is an empty shell. The two charts side-by-side with 23 candles and EMA/floor/ceiling lines on essentially flat intraday BTC will not help a trader decide anything. The Short Side Watch panel showing "INACTIVE" is the most actionable thing on the page.

---

### Trading Ledger (ledger.html)

**Data source:** Pure static JavaScript arrays (`week2Trades`, `week1Trades`) — NO JSON feed  
**Libraries:** IBM Plex Mono + Chakra Petch (Google Fonts)

#### Working ✅
- Week 1 card (12W/3L) renders — 15 trades displayed correctly from hardcoded array
- Equity curve SVG renders — tooltip on hover works
- Card toggle (expand/collapse) works
- Trade row coloring (green left border = win, red = loss) works
- Grade badges styled correctly
- Mobile responsive (column hiding at 600/375px)
- Footer date renders dynamically

#### Broken ❌
1. **Week 2 has only 3 of 7 trades** — hardcoded array shows:
   - Trade 1: `KXBTCD-26APR0317-T67299`, 114×, $57 cost, $57 profit, 100% ROI
   - Trade 2: `KXBTCD-26APR0317-T68299`, 116×, $58 cost, $58 profit, 100% ROI  
   - Trade 3: `KXBTCD-26APR0317-T69299`, 115×, $57.81 cost, $57.19 profit, 98.9% ROI
   
   **Missing 4 trades entirely:** ETH $2,080 (+$94.05), ETH $2,120 (+$61.95), ETH $2,160 (+$60.95), ETH $2,200 (+$49.29)
   
   **BTC amounts also wrong:** Actual confirmed trades are $68,300 (+$53, cost $147.96), $69,800 (+$98.40, cost $195.04), $71,300 (+$81.50, cost $152.90) — the ledger has wrong tickers, wrong costs, and inflated ROI (100% vs actual 36-53%)

2. **Stats bar is stale with wrong numbers:**
   | Field | Current | Correct |
   |-------|---------|---------|
   | Total Trades | 18 | 22 |
   | Win-Loss | 15–3 | 19–3 |
   | Win Rate | 83.3% | 86.4% |
   | Total P&L | +$824.97 | +$1,151.99 |
   | Portfolio | $1,587 | $1,681.78 |
   | Return | +76.4% | +86.9% |

3. **Contract tickers are unreadable** — `KXBTCD-26APR0317-T68299` appears in the Contract column. A human trader cannot instantly read this. No parsing logic to display "BTC $68,300 YES" or "ETH $2,080 YES".

4. **Week 2 header shows wrong totals:**
   - Current: "3–0 · +$172.19 · Avg ROI: 99%"
   - Should be: "7–0 · +$499.21 · Avg ROI: 62%"
   - Week totals footer: Deployed $172.81, Returned $345.00 — both wrong

5. **Equity curve wrong** — bars are generated from the wrong/incomplete trade data. The Week 2 portion will show 3 ~100% ROI bars, not the 7 varied-ROI bars from actual history.

#### Missing 🔇
6. **Not driven from JSON** — trade data is hardcoded in JS. Agent would need to write to a `data/ledger_data.json` or this file would need manual editing every time.
7. **No cumulative equity curve** — shows P&L per trade but not the portfolio balance line over time (900 → 900+x → etc.)

**UX Grade: 5/10**

> The ledger's visual design is the best of the four pages — week cards, equity curve, grade badges, hover tooltips all work well. The problem is the data is wrong. Week 2 shows 3 trades with inflated numbers instead of 7 real trades with real ROI. A trader checking performance against actual Kalshi history would immediately notice the discrepancy and lose trust in the whole board.

---

### Architecture (architecture.html)

**Data source:** Pure static HTML — no data feed  
**Libraries:** IBM Plex Mono + Chakra Petch

#### Working ✅
- Bot fleet descriptions are accurate: Eagle (retail phase, CVD), Nano (perp structure, funding), Osprey (whale flow, zone position), Vulture (EMA stack, RSI), Weatherman (F&G, macro)
- Symphony Protocol flow diagram matches actual system (7+/9 threshold)
- Morfeus Engine description is substantially accurate
- Navigation (sticky nav, anchor links) works
- Flow diagrams render correctly
- Mobile responsive (grid collapses to 1-col at 640px)
- Overall information is dense and accurate for a reference document

#### Outdated / Inaccurate ⚠️
1. **Version says "Mar 28, 2026"** — 4 days old, needs update
2. **"93% hit rate on EXECUTE-filtered signals"** in the overview flow diagram — actual measured win rate is 86.4% (22 trades) which includes Week 2's lower-ROI ETH positions. The 93% claim is unverified.
3. **Morfeus Layer 3 says "Claude interpretation"** — the analyst routing was changed from Anthropic Claude to Gemini (see `agent.py` fix from this session). Architecture says "Claude interpretation" but the system now calls `call_gemini()`.
4. **Symphony table says "wake Claude Code"** at SYMPHONY_OVERRIDE — still accurate (Claude Code CLI is still used for decision-making), but the analyst within the agent is now Gemini.
5. **PROPHECY tier description not present** — the architecture doesn't document the PROPHECY/RED_PILL/GOLD tier system or the direction-specific thresholds (OVER ≥92% + base≥30% vs UNDER ≥85%) that were recently fixed.
6. **Nano 3-tier logic not documented** — the 7/5-6/<5 ACCUM threshold split (full override / structural check / full kill) isn't in the docs.
7. **No mention of Construct dedup system** or the 45s dedup cache.
8. **MCP tools section** (referenced in nav) — need to verify if this section reflects current tools 23-25 (get_portfolio, get_positions, get_orders).

#### Missing 🔇
9. **No data flow diagram showing dashboard_data.json** — how the agent writes to the file, what fields it populates, and which pages read which fields isn't documented.
10. **No trading protocol section** — Ping Pong rules, PROPHECY trade sizing, when to NOT trade.

**UX Grade: 7/10**

> Architecture is the most technically complete page. A new operator could read it and understand the system. The outdated version number and "93%" claim are the most egregious issues. The missing Gemini reference and undocumented recent fixes are also gaps but won't confuse a regular user.

---

## DATA FLOW AUDIT

### dashboard_data.json — Current Status

| Field | Status | Value | Notes |
|-------|--------|-------|-------|
| `updated_at` | ✅ Fresh | 2026-04-01T16:14:39Z | ~14 min old at time of audit |
| `balance.available` | ✅ | $1,681.78 | Correct |
| `balance.portfolio_value` | ❌ Wrong | 0.0 | Should be ~$1,681.78 or sum of position market values |
| `balance.deployed` | ❌ | 0 | Should reflect deployed capital |
| `positions` | ⚠️ Empty | [] | May be correct if no open positions |
| `sell_orders` | ⚠️ Empty | [] | May be correct |
| `buy_traps` | ⚠️ Empty | [] | May be correct |
| `top_edges` | ❌ Garbage | 10× DOGE BUY_NO at 99¢ | See below |
| `price_history.BTC.series` | ⚠️ Thin | 24 points (12h) | Not enough for charting context |
| `price_history.ETH.series` | ⚠️ Thin | 24 points (12h) | Same |
| `fleet_summary` | ✅ | 11 assets | Full data including SOL, XRP, DOGE, BNB, HYPE, TAO, ZEC, LINK, BCH |
| `symphony` | ✅ | SYMPHONY_OVERRIDE 91% | Correct |
| `macro` | ✅ | F&G=8, Wthr=7, STRONG LONG | Current |
| `catalysts` | ✅ | 5 events through Apr 3 | JOLTS (past), ADP, ISM Mfg, Tariff Day, NFP |

**Missing fields:**
- `daily_pick` — no field exists for the daily trade recommendation
- `weekly_picks` — no field exists
- `analyst_notes` / `gemini_summary` — no analyst output field
- `portfolio_pnl` — no realized P&L field
- `price_history` only stores today's data — no historical price data for meaningful charting

### top_edges — Root Cause Analysis

All 10 entries in `top_edges` are identical except for the DOGE strike:
```
asset: DOGE, direction: UNDER, signal: BUY_NO, 
confidence: 91, tier: PROPHECY, mkt_price: 99.0, edge_pct: 96.5, days: 0.37
```

**What happened:** The edge scanner (`kalshi_edge_scanner.py` or `contract_scanner.py`) is running PROPHECY-level predictions on DOGE UNDER contracts expiring in ~9 hours. DOGE is at $0.0927 — all strikes from $0.005 to $0.05 are deep OTM (NO wins if DOGE stays above strike). The "edge" is trivially obvious — of course DOGE won't drop 95%. The system is correctly computing that these NOs are nearly certain to win, but it's showing this as "PROPHECY BUY_NO" which communicates nothing useful.

**Fix location:** Two places:
1. `contract_scanner.py` / `kalshi_edge_scanner.py` — needs post-processing filter: exclude any contract where `mkt_price > 0.70` (already priced in, no ROI); exclude if `signal == 'BUY_NO'` and the operator doesn't trade NO positions.
2. `dashboard_data.json` writer in `agent.py` — needs to filter `top_edges` before writing to JSON

### Data Update Frequency

Based on `updated_at` timestamps in price_history.series:
- Price history points are ~30 minutes apart
- JSON updated at least every 30 minutes (agent cycle)
- Catalysts and macro fields: updated each cycle (not real-time F&G)

---

## PRIORITIZED OVERHAUL PLAN

### Phase 1 — CRITICAL (Do First — these are wrong/broken data)

**1. Fix ledger.html Week 2 trade data** — `ledger.html:227–231`
- Replace 3 wrong BTC trades with correct 7 trades (4 BTC + 4 ETH from confirmed Kalshi history above)
- Fix all costs, payouts, profits, ROI, and tickers
- Update stats bar: 22 trades, 19W/3L, 86.4%, +$1,151.99, $1,681.78, +86.9%
- Update Week 2 header: 7-0, +$499.21, Avg ROI 62%, Mar 31 – Apr 6 2026
- Update Week 2 totals footer: Deployed $809.63, Returned $1,308.84
- **Effort:** 30 min | **File:** `ledger.html`

**2. Fix portfolio_value in JSON** — `agent.py` dashboard writer
- `balance.portfolio_value: 0.0` causes "$0" to display on both Signal Board and Ping Pong nav
- Fix: write `portfolio_value` as `balance.available + total_position_market_value` or pull from `get_balance()` directly
- **Effort:** 15 min | **File:** `agent.py` (dashboard_data writer function)

**3. Fix edge scanner — filter out garbage contracts** — `contract_scanner.py` or wherever `top_edges` is built
- Filter 1: `mkt_price > 0.70` → exclude (already priced in, no tradeable ROI on YES)
- Filter 2: `signal == 'BUY_NO'` on non-DISTRIBUTION fleet → exclude (operator trades YES longs)
- Filter 3: `days < 0.1` → exclude (< 2.4h to expiry, no time to trade)
- Only show contracts in 5¢–50¢ YES range with genuine probability gap
- **Effort:** 20 min | **Files:** `markets/kalshi/contract_scanner.py` + edge scanner output pipeline

**4. Fix contract label display in ledger** — `ledger.html:270–272`
- Parse ticker to human-readable: `KXBTCD-26APR0317-T68299` → `BTC $68,300 YES`
- Regex: extract asset (BTC/ETH), strike from `-T{strike}`, direction from side field
- Display as `<span class="ticker">BTC $68,300 YES</span>` in Contract column
- Keep raw ticker in small text below for reference
- **Effort:** 45 min | **File:** `ledger.html` (add `parseTicker()` helper)

---

### Phase 2 — IMPORTANT (Do Second)

**5. Add Daily Pick section to Signal Board** — `index.html:96`
- Add new section between Edge Scanner and Macro: `DAILY PICK — GEMINI RECOMMENDATION`
- Reads from `daily_pick` field in dashboard_data.json (needs to be added to JSON by agent)
- Shows: Asset, Direction, Entry zone, Confidence tier, Reasoning (1-2 sentences)
- Show "NONE — WAIT" prominently if no pick
- **Effort:** 2h (agent write + UI) | **Files:** `index.html`, `agent.py`

**6. Add Weekly Picks / Avoid List section** — `index.html`
- Below Daily Pick: collapsible section "WEEK OF APR 1–6"
- Top 3 setups + Top 3 avoid signals
- Driven from `weekly_picks` JSON field (agent writes once per week)
- **Effort:** 3h | **Files:** `index.html`, `agent.py`

**7. Backfill price history** — `agent.py`
- Currently writes only intraday price history (~12h lookback)
- Extend to 48h minimum (96 data points at 30-min intervals)
- Store in rolling buffer: keep latest 96 points per asset
- Charts become meaningful with 48h of BTC context
- **Effort:** 1h | **File:** `agent.py`

**8. Drive ledger from JSON data file**
- Create `data/ledger_data.json` with weeks array
- `ledger.html` fetches this instead of using hardcoded arrays
- Agent writes new trades to ledger_data.json when fills are detected
- **Effort:** 3h | **Files:** `ledger.html`, `agent.py`

**9. Verdict card on Signal Board**
- Add "TRADE SIGNAL" box at top of page (below snapshot, above fleet grid)
- Shows: Current best setup with BUY / WAIT / AVOID and one-line reason
- Populated from `daily_pick` field
- If SYMPHONY_OVERRIDE + STRONG LONG + pick exists → green highlight box
- **Effort:** 2h | **Files:** `index.html`, requires Phase 2 #5

---

### Phase 3 — NICE TO HAVE

**10. Dashboard redesign — verdict card** — `dashboard.html`
- Add "CYCLE STATUS" summary bar at top: current phase, recommended action, confidence
- Format: `BTC | NEUTRAL | HOLD — no CAPITULATION + ACCUM combo active | Watch $69,285 ceiling`
- Position: between nav bar and charts
- **Effort:** 2h | **File:** `dashboard.html`

**11. Position-aware proximity alerts** — `dashboard.html` + `index.html`
- When `positions` is non-empty: show banner "BTC is $X from your $Y strike — [above/below]"
- Reads `pos.strike` vs `btc.price`
- **Effort:** 1h | **Files:** both HTML files (reads same positions field)

**12. Historical equity curve on ledger**
- Add second equity curve showing running portfolio balance over time (not per-trade P&L)
- $900 → $985 → $1,037 → ... → $1,681.78
- More meaningful for performance visualization
- **Effort:** 2h | **File:** `ledger.html`

**13. Architecture.html accuracy pass** — `architecture.html`
- Update version to v1.1, date to Apr 1 2026
- Change "Claude interpretation" → "Gemini interpretation" in Morfeus Layer 3
- Remove "93% hit rate" claim or replace with "86.4% confirmed win rate (22 trades)"
- Add PROPHECY tier documentation (direction-specific thresholds)
- Add Nano 3-tier threshold table
- Add dashboard_data.json data flow diagram
- **Effort:** 1.5h | **File:** `architecture.html`

**14. Mobile optimization — Signal Board**
- Fleet grid currently has 9 columns on mobile — too wide at 375px
- Hide CVD and EMA% columns at ≤480px, keep Asset/Eagle/Nano/RSI/Trend
- **Effort:** 30 min | **File:** `index.html`

---

## REDESIGN RECOMMENDATIONS

### Signal Board (index.html)
**Keep:** Fleet grid layout, macro section, catalyst timeline, symphony banners  
**Redesign order:** Move snapshot bar to show Available Cash / Realized P&L / Win Rate (not portfolio_value which is always $0). Move Daily Pick to top of page — it should be the FIRST thing a trader sees. Replace Edge Scanner with "Top Setups" that only shows YES contracts in 5¢–50¢ range with probability gap.

### Ping Pong Dashboard (dashboard.html)
**Full redesign scope:** The page needs a "War Room" header row showing: Cycle Phase · BTC Price vs EMA200 · ETH Price vs EMA200 · Symphony State · Cash Available. Charts need 48h data to be useful. Add a VERDICT section between charts and cycle tracker: a simple color-coded decision block (GREEN = enter, YELLOW = watch, RED = wait/exit). The cycle tracker is already well-designed — when positions exist it will be powerful.

### Trading Ledger (ledger.html)
**Keep:** Week card structure, equity bar chart, grade badges, hover tooltips — all good UX  
**Change:** Fix the data (priority 1). Add JSON data feed. Add ticker parsing. Update the weekly totals footer to auto-compute from trade data rather than hardcoding.

### New Features (highest ROI)
1. **Daily Pick** — one field in JSON, one section in index.html, massive UX improvement
2. **Portfolio Value fix** — one line in agent.py, fixes display across two pages
3. **Edge filter** — 20 minutes of code eliminates an entire garbage section

---

## APPENDIX — JSON FIELDS NEEDED BUT MISSING

| Field | Used By | Agent Source |
|-------|---------|--------------|
| `daily_pick` | index.html, dashboard.html | Gemini analyst output |
| `weekly_picks` | index.html | Agent weekly summary |
| `balance.portfolio_value` | index.html snap, dashboard nav | portfolio_reader.get_balance() |
| `price_history[*].series` (48h depth) | dashboard charts | Agent price accumulator |
| `analyst_notes` | index.html (future) | Gemini analyst output |
| `realized_pnl` | ledger, snapshot | portfolio_reader.get_fills() |

---

*Generated April 1, 2026 — All data from live `dashboard_data.json` read + full HTML source code review.*
