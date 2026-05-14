# Morning Trading Playbook — Dry-Run v1

*Paste this into `github.com/Dylanxmt/finance-playbooks/morning-dryrun-v1.md`. Version: 2026-04-22 (r2, post-simulation patches + EOD signal integration).*

*This file is fetched at runtime by the Anthropic scheduled trigger. No orders will be placed by any workflow that loads this file — all trade proposals exist only as rationale in the Gmail draft for Dylan's review.*

---

## 🔴 CRITICAL — DRY-RUN MODE RULES

1. You MUST NOT call any order-execution tools: `place_stock_order`, `place_crypto_order`, `place_option_order`, `cancel_order_by_id`, `replace_order_by_id`, `close_position`, `close_all_positions`, `exercise_options_position`.
2. Every "trade" you propose exists only in the Gmail draft. You are simulating execution, not performing it.
3. If you catch yourself about to execute, stop immediately. Finalize the email draft with proposed trades and exit.
4. Read-only tools are fine: all `get_*` Alpaca tools, `get_stock_snapshot`, `get_stock_latest_quote`, etc.

---

## Context injected by trigger

- **Mode:** DRY_RUN
- **Target allocation:** 70% invested / 30% cash (hard floor: 25% cash — never breach)
- **Account:** Alpaca paper `{{ACCOUNT_ID}}` (provided in trigger context)
- **Recipient:** `{{RECIPIENT_EMAIL}}` (provided in trigger context)
- **Today:** the date the trigger fires

## Risk-tier reference (use for sizing trail stops)

| Tier | Trail % | Examples |
|------|---------|----------|
| Stable/defensive | 3% | XLP, XLU, JPM, XLF, XLV, KO, PG, VZ |
| Moderate | 4% | SPY, IWM, GLD, HD, XLE, XLI, XLB, XLK, sector ETFs generally |
| Volatile/growth | 5% | NVDA, META, FCX, AMD, TSLA, small caps, momentum single-names |

## Hard caps (enforced in Step 7)

- Max per single trade: **5% of equity**
- Max new positions opened per day: **3**
- Min cash reserve: **25% of equity** (never breach, even for top-ups)
- Order types: **LIMIT only** — day or gtc, never market
- Every new position described in the email must include a corresponding trail-stop specification (ticker, qty, trail %, initial stop estimate)

---

## Step 0 — Kill Switch Check (two paths, either engaged → standdown)

Two independent kill paths. Check the repo file first (primary), then Gmail label (secondary). Either engaged → standdown.

**Step 0a — Repo kill-switch file (primary path).**
1. Fetch via Bash + curl (10s timeout):
   ```
   curl -s --max-time 10 https://raw.githubusercontent.com/Dylanxmt/finance-playbooks/main/kill-switch/state.txt
   ```
2. If fetch succeeds: parse line 1 (trimmed of whitespace).
   - If line 1 is exactly `PAUSED`: standdown immediately. Create draft titled `Morning Brief — [Date] — STANDING DOWN (repo kill switch ENGAGED)`. Body: include the full file contents so Dylan sees the metadata footer (who set it, when, why). Add a line "Disengage at: https://github.com/Dylanxmt/finance-playbooks/blob/main/kill-switch/state.txt". Exit.
   - If line 1 is `ARMED` or anything else: proceed to Step 0b.
3. If fetch fails (non-zero exit, empty response, network error): sleep 5s, retry once.
4. If both fetch attempts fail: set `repo_kill_reachable = false`, log the failure, proceed to Step 0b.

**Step 0b — Gmail label (secondary path).**
5. Search Gmail for `label:TRADING-PAUSED newer_than:2d`.
6. If ANY match found:
   - Standdown. Create draft titled `Morning Brief — [Date] — STANDING DOWN (Gmail kill label active)`.
   - Body: one line stating "Kill switch detected via Gmail label. Remove the `TRADING-PAUSED` label to resume."
   - Exit.
7. If the Gmail search itself errors: set `gmail_kill_reachable = false`, log the failure, continue.

**Step 0c — Fail-closed if both paths unreachable.**
8. If `repo_kill_reachable = false` AND `gmail_kill_reachable = false`: stand down. Subject `Morning Brief — [Date] — STANDING DOWN (both kill paths unreachable — fail-closed)`. Body explains both failures so Dylan can investigate before next run.
9. If either path is reachable and reports armed/no-label, proceed to Step 1.

## Step 1 — Health Check (discovery-tolerant + degraded-mode fallback)

The Alpaca MCP connector lives behind two layers: (a) the Render-hosted MCP server itself, and (b) the Anthropic agent runtime's discovery / tool-registration layer. UptimeRobot keeps the Render dyno warm (verified 2026-04-27 — initialize handshake responds in ~140ms), so the dominant failure mode is **runtime discovery taking longer than the first attempt**, not Render cold-start.

**Updated 2026-05-10 (post-May-8 standdown):** Discovery uses exponential backoff (not flat retries), probes Render at the attempt-3 boundary to distinguish "Render down" from "runtime slow," and routes discovery failures to a Yahoo-backed **degraded mode** (Phase D) instead of producing a zero-visibility standdown email. True standdowns are reserved for account-level halts, market-closed days, and the no-fallback-data case.

**Phase A — wait for discovery (priming + exponential backoff).**
1. Sleep 45 seconds before any Alpaca call. Connector tools may not be registered in the runtime's tool index until discovery completes; calling them too early returns "no matching tools" and burns the budget.
2. Run up to 7 ToolSearch attempts. Each attempt: query `"select:mcp__Alpaca-Trading__get_clock,mcp__alpaca__get_clock"`, `max_results: 2` — these are the two known Alpaca connector prefix conventions (cloud HTTP connector uses `mcp__Alpaca-Trading__*`, local stdio uses `mcp__alpaca__*`). If EITHER tool schema is returned, capture which prefix matched as `alpaca_tool_prefix` (use it for ALL subsequent Alpaca calls in this run — every `mcp__alpaca__*` reference later in this playbook should be read as the captured prefix), and proceed to Phase B.

   **DO NOT fall back to keyword search** (`"Alpaca"`, `"trading"`, `"stock order"`, etc.). Keyword ToolSearch cannot return tools that have not completed runtime registration — it reports 0 matches and burns the budget without ever succeeding. Only `select:` queries against exact tool names can find a registered-but-not-yet-indexed connector. If both candidate prefixes return empty across all 7 attempts, the connector has not registered for this run — go to Phase D.
3. Backoff schedule (sleep AFTER a failed attempt, BEFORE the next):
   - After attempt 1: sleep 10s
   - After attempt 2: sleep 20s
   - After attempt 3: **PROBE RENDER** (see sub-step below), then sleep 40s if continuing
   - After attempt 4: sleep 60s
   - After attempt 5: sleep 60s
   - After attempt 6: sleep 60s
   - After attempt 7: stop. Set `alpaca_available = false`, `failure_class = "discovery_exhausted"`. Go to Phase D.
4. Discovery budget: 45s priming + ~250s retry = ~5 min worst case. Agent fires 9:00 AM ET, market opens 9:30 AM → ~25 min headroom remains for Steps 2-12 even at worst-case.

**Render probe (sub-step, runs once after attempt 3 fails):**
Use `WebFetch` on `https://alpaca-mcp-server-8zw5.onrender.com/` with a 15s timeout. Interpret:
- Any HTTP response (200, 404, 405, 400, 500, etc.) → set `render_status = "up"`. The server is live; runtime hasn't finished tool negotiation. KEEP retrying through attempt 7.
- Connection refused, DNS error, timeout, or 502/503/504 → set `render_status = "down"`. Stop retrying — set `alpaca_available = false`, `failure_class = "render_down"`. Skip remaining attempts. Go to Phase D.
- If `WebFetch` is unavailable in this runtime, set `render_status = "unprobed"` and continue retries on the schedule above.

Capture and persist for the email body: `discovery_attempts_made` (1-7), `render_status` ("up" / "down" / "unprobed" / "n/a"), `failure_class`.

**Phase B — probe Alpaca.**
5. Call `get_clock` using the `alpaca_tool_prefix` captured in Phase A (i.e., `{alpaca_tool_prefix}__get_clock`). Allow up to 30s.
6. If it fails or times out, wait 30s and retry once.
7. If both attempts fail *despite tools being registered*: this is a separate failure class — Alpaca API or auth, not connector discovery. Set `alpaca_available = false`, `failure_class = "alpaca_api_error"`. Capture the error text from each attempt. Go to Phase D.

**Phase C — verify account.**
8. Call `get_account_info`. Verify:
   - `status == "ACTIVE"`
   - `trading_blocked == false`
   - `account_blocked == false`
   - `trade_suspended_by_user == false`
9. If any check fails: this is a deliberate halt (account itself is blocked) — go to **true standdown**, NOT Phase D. Subject: `Morning Brief — [Date] — STANDING DOWN (account [which check failed])`. Body includes which check failed and the raw account JSON. Exit.
10. Call `get_calendar` for today. If market is closed the whole day (holiday), true standdown with subject `Morning Brief — [Date] — Market closed`. Exit.
11. Record baseline metrics: `equity`, `cash`, `last_equity`, `portfolio_value`, `buying_power`, `non_marginable_buying_power`. Set `alpaca_available = true`, `position_source = "alpaca-live"`. Proceed to Step 1.5.

**Phase D — degraded-mode fallback (replaces pure standdown for discovery/probe failures).**
Entered only when `alpaca_available = false` from Phase A or B. Phase C account-block / market-closed failures bypass this and go to true standdown.

12. Search Gmail: `in:draft from:{{RECIPIENT_EMAIL}} subject:"EOD Report" newer_than:5d`. Take the most recent match (yesterday's, in the typical case — extends to 5 days for long weekends / holiday gaps). **The `in:draft` flag is load-bearing** — agent reports are saved as drafts (never sent), and the Gmail connector's `search_threads` tool excludes drafts by default. Without `in:draft`, this query returns empty even when the EOD report exists.
13. Call `get_thread` with `messageFormat: FULL_CONTENT`. Parse the EOD body for:
    - Portfolio table rows: ticker, qty, avg cost, close price.
    - Account summary line: `equity`, `cash`, `buying_power` — label these "as of [EOD date]".
    - The `<!-- SIGNAL_RESULT_START ... SIGNAL_RESULT_END -->` block (if present). This carries straight into Step 1.5's `active_signals[]` — the signal feed is independent of Alpaca.
14. For each ticker, fetch a current premarket/opening quote:
    - First try `mcp__yahoo-finance__get_stock_price`.
    - If that fails or the MCP is unavailable, try `mcp__financial-datasets__get_current_stock_price`.
    - If both fail for a ticker, mark that row `price_unavailable` and continue.
15. Compute approximate per-position values:
    - `position_value ≈ qty × current_price`
    - `unrealized_pl ≈ (current_price − avg_cost) × qty`
    - `overnight_change_pct ≈ (current_price / eod_close − 1) × 100`
16. Compute approximate portfolio total: `approximate_equity ≈ sum(position_values) + cash_from_eod`. Mark `†` and footnote "approximate — Alpaca offline, computed from yesterday's EOD close + premarket quotes. Cash and buying power frozen as of [EOD date]."
17. Set `position_source = "fallback (yesterday's EOD + premarket quotes)"`. **DEGRADED-MODE CONSTRAINTS:**
    - Step 1.5 (signal carry-forward) STILL RUNS — signals are independent of Alpaca. Use the EOD email already fetched in step 12.
    - Skip Step 2's overnight position diff (no live positions to compare; use the fallback data instead).
    - Step 3 market context still runs (web-based).
    - Step 4 position-level scan: use Yahoo for the per-ticker price and news; SKIP the trail-stop headroom calculation (no live stops visible). Note "stop headroom unavailable in degraded mode."
    - Step 5 sector rotation: attempt via Yahoo for the 11 sector ETFs (XLK, XLF, etc.); if Yahoo fails for ≥3 of them, omit the sector-rotation section.
    - **Skip Step 6 (opportunity scan) and Step 7 (guardrail filter) entirely.** No new-position generation in degraded mode — no live cash, no order plumbing, no way to verify the math.
    - Step 8 (decision) lands in "STAND DOWN — degraded mode, monitoring only" automatically.
    - Step 10 (email) renders an info-only report with the Connectivity Status block.
18. If no recent EOD email is found OR all current-price lookups fail: this is a **true standdown**. Subject: `Morning Brief — [Date] — STANDING DOWN (degraded fallback also failed: [reason])`. Body explains what was tried (EOD email search query and result, ticker count attempted, MCP availability). Exit.

**Total budget worst case:** ~5 min Phase A + ~60s Phase B + ~30s Phase C + ~90s Phase D = ~8 min. Agent fires 9:00 AM ET, market opens 9:30 AM → still ~22 min of headroom even at degraded-mode worst case.

## Step 1.5 — Active Signal Watchlist (from EOD research)

The EOD agent runs structured signal research each weekday and embeds a machine-readable block at the bottom of its email. Load the most recent one so morning decisions benefit from last night's research.

1. Search Gmail: `in:draft from:{{RECIPIENT_EMAIL}} subject:"EOD Report" newer_than:3d` — take the most recent matching message. Call `get_thread` with `messageFormat: FULL_CONTENT` to retrieve the body (snippets are truncated and will miss the SIGNAL_RESULT block). **The `in:draft` flag is required** because reports are persisted as drafts; Gmail search excludes drafts by default.
2. Parse the body for the block delimited by `<!-- SIGNAL_RESULT_START` and `SIGNAL_RESULT_END -->`.
3. Extract fields: `DECISION`, `CHANNEL`, `SIGNAL_ID`, `SIGNAL_NAME`, `PERFORMANCE`, `CREATIVITY`, `CHECK_RULE`, `FINDING_SUMMARY`. Each field is on its own line in `LABEL: value` format — no embedded newlines inside a value.
4. Determine active signals:
   - `DECISION == AUTO-PROMOTE` → **active** — its `CHECK_RULE` applies during Step 6 (opportunity scan) and must appear in the email's "Active Signals" section.
   - `DECISION == AUTO-DROP` → ignore, do not display.
   - `DECISION == NEEDS_REVIEW` → **not active** — note it once in the email under "Signals pending review" but do NOT let it gate any trade.
5. Also scan the same email for `SIGNAL_RESULT` blocks from the prior 2 days (use `newer_than:7d` as widening) to maintain a rolling set of promoted signals. If a signal has been superseded by a newer result for the same `CHANNEL`, keep only the latest.
6. If no EOD email is found within 3 days, record `signal_feed_status = stale` and proceed — the morning agent can still operate, but the email must say "No fresh EOD signal research available" in the Active Signals section.

Hold the parsed result set in working memory as `active_signals[]` and `pending_signals[]`. These feed Steps 6 and 10.

## Step 2 — Portfolio Snapshot

Gather:
- `get_all_positions` → full list with qty, avg_entry_price, current_price, market_value, unrealized_pl, unrealized_plpc
- `get_orders` with `status: open` → all working orders (trail stops, pending buys, etc.)
- `get_orders` with `status: closed`, `after: <previous calendar day 00:00 ET>` → any overnight fills (gapped stops, etc.)
- `get_account_activities` filtered to `FILL` from the previous trading day → authoritative exit/fill list for overnight reconciliation

Compute:
- **Overnight position diff**: compare the current `get_all_positions` set to yesterday's EOD snapshot (from the most recent EOD email body). Any ticker present yesterday but missing today = **overnight stop-out** — record ticker, qty, exit price, approx P/L. These go in the email's "Overnight Events" line at the top.
- **Cash %** = cash / equity
- **Invested %** = position_market_value / equity
- **Largest position %** = max(market_value) / equity — flag if >12%
- **Stops-at-risk**: positions where `(current_price - stop_price) / current_price < 0.02` → within 2% of stop
- **Earnings proximity**: for each holding, run a `WebSearch` for `"{TICKER} Q[1-4] earnings date 2026"` — flag any position with earnings within 5 trading days with `🔔 EARN`. (Yahoo/financial-datasets MCPs aren't enabled in the trigger context; WebSearch is reliable enough for date lookups.)
- **Sector map** (assign each holding to a category):
  - Core index: SPY, VOO, IVV
  - Large-cap tech: NVDA, META, MSFT, GOOGL, AMZN, AAPL, AMD, TSLA
  - Small cap: IWM, IJR
  - Financials: XLF, JPM, BAC, KRE
  - Healthcare: XLV, UNH, JNJ
  - Industrials: XLI, CAT, GE
  - Consumer disc: HD, XLY, AMZN
  - Consumer staples: XLP, KO, PG
  - Energy: XLE, XOM, CVX
  - Materials: XLB, FCX, NEM
  - Utilities: XLU, NEE
  - Real estate: XLRE
  - Safe haven: GLD, SLV, TLT
  - Communication: XLC, GOOGL, META

## Step 3 — Market Context

Pull snapshots for: SPY, QQQ, IWM, TLT, GLD, DXY (or UUP as proxy). If VIX or VIXY is available, include it.

Fetch news via `WebSearch` (Yahoo/TradingView MCPs aren't enabled in the trigger context). Run TWO searches in parallel and synthesize:
1. `WebSearch: "stock market news today {YYYY-MM-DD}"` — broad market summary
2. `WebSearch: "{specific catalyst}"` ONLY if Step 1 surfaces a named event (Fed meeting, CPI print, earnings cluster, geopolitical event) — drill in for context

If Step 1 returns nothing useful, record `news_feed = empty` and note in the email: "No fresh market news retrievable — tape read is price-only."

Future migration path: if richer data is needed, enable Yahoo Finance / TradingView / FRED MCPs in the trigger config and update this step accordingly.

Derive 3–5 plain-English signals. Examples of what "a signal" looks like:
- "10yr yield -12bp overnight on soft CPI print (2.8% vs 3.0% est)"
- "NVDA +2% pre-market after semi earnings beat"
- "Gold +0.8% on a risk-on day — institutions hedging"
- "Dollar breakout to 6-month highs — headwind for multinationals"

If nothing material is happening, note that explicitly: "quiet tape, no fresh catalysts."

## Step 4 — Position-Level Scan

For each existing holding, check:
- Latest quote vs. trail stop → headroom %
- News in last 24h via `WebSearch: "{TICKER} stock news {YYYY-MM-DD}"` — only run for positions with notable price moves (>2% intraday) or known event proximity; skip for boring positions to save search budget
- Earnings within 5 trading days — reuse the earnings dates already gathered in Step 2's earnings-proximity sweep (do NOT duplicate the search)

Flag to the email:
- Any position within 2% of its stop ("at-risk")
- Any position >12% of equity ("trim candidate")
- Any position with material news (earnings, guidance, downgrade, etc.)
- Any position with earnings within 5 trading days (`🔔 EARN`)

## Step 5 — Sector Rotation Analysis

1. Call `mcp__alpaca__get_stock_snapshot` with `symbols: "XLK,XLF,XLV,XLI,XLY,XLP,XLU,XLE,XLRE,XLB,XLC"` in a single request.
2. For each sector, compute **1-day percent change** = `(dailyBar.c - prevDailyBar.c) / prevDailyBar.c`.
3. Rank sectors best-to-worst; include the full ranked list in the email under "Sector Rotation (1-day)" — readers can infer leadership/laggards at a glance.

Compute these specific **pair deltas** (leader % change minus laggard % change) and interpret:

- **XLY vs XLP** — consumer confidence barometer (XLY > XLP → risk-on)
- **XLI vs XLU** — growth vs defensive (XLI > XLU → pro-cyclical)
- **XLF vs XLRE** — rate sensitivity (XLF > XLRE → rates firm, growth financial-friendly)
- **QQQ vs SPY** — tech leadership (compute same way using QQQ/SPY snapshots)
- **IWM vs SPY** — small-cap risk appetite

Interpret briefly in the email:
- XLY > XLP by >0.5% → risk-on, growth tilt confirmed
- XLP > XLY by >0.5% → defensive rotation, caution signal
- IWM > SPY → broad risk appetite expanding
- QQQ leading by >1% → tech-driven tape, watch for rotation fatigue
- All pairs inverting together (XLP/XLU/XLRE leading) → defensive regime, lean toward trims/stand-down

## Step 6 — Opportunity Scan

Using signals from Steps 1.5 and 3-5 and portfolio analysis from Steps 2 and 4, generate candidate actions across all four authorities.

**Apply active EOD signals (from Step 1.5):** for each candidate, run it through every active `CHECK_RULE` in `active_signals[]`. The `CHECK_RULE` acts as a gate or confirm:
- If the rule applies to this ticker/sector AND returns "confirmation" → strengthen conviction, record in rationale's "Signal confirmation" line
- If the rule applies AND returns "rejection" → discard the candidate, log under Step 7 rejections citing the signal name
- If the rule applies AND returns "neutral" / insufficient data → proceed but note "signal inconclusive" in rationale
- If the rule does not apply to this ticker/sector → ignore

Example: if the active signal has `SIGNAL_ID=SW-003` and `CHECK_RULE="Require XLY/XLP > 2.15 for discretionary entries; reject if < 2.00"`, then any XLY/HD/AMZN candidate gets gated on the current ratio before advancing.

**(a) Open new position.** Criteria: sector gap in portfolio + catalyst in that sector + ticker aligned with risk tier. Prefer sector ETFs for breadth; single names only with strong conviction + clear thesis.

**(b) Top up existing position.** Criteria: holding with fresh positive catalyst, unrealized P/L ≥ 5%, trail stop with ≥3% headroom, existing position <10% of equity post-topup. Don't top up a name that's within 3% of its trail stop.

**(c) Trim concentrated winner.** Criteria: any position >12% of equity. Propose trim back to ~10%. Compute the exact share count to sell. Adjust trail stop proportionally (cancel + replace in the email plan, do not actually cancel in dry-run).

**(d) Rebalance toward target.** If cash > 35%: lean into deploying (more bias to (a)/(b)). If cash < 28%: lean into trimming (more bias to (c)). Use 70/30 as anchor.

Generate up to **5 candidate trades** across (a)-(d). Each candidate must have:
- Authority label (a/b/c/d)
- Signal it's responding to
- Proposed ticker + qty + limit price + est. cost
- Sector/purpose in portfolio
- Risk tier for trail stop

## Step 7 — Guardrail Filter (HARD)

Guardrails are checked in two passes:

**Pass 1 — Per-candidate checks** (applied to each candidate independently):

- [ ] Est. cost ≤ 5% of equity
- [ ] Order type = LIMIT (never market)
- [ ] Trail-stop risk tier assigned (3/4/5%)
- [ ] No duplicate open buy order for this ticker already working
- [ ] Ticker did not stop out in the last 5 trading days (to avoid re-entering a just-broken thesis) — include overnight stop-outs detected in Step 2
- [ ] No earnings for this ticker in the next 5 trading days (unless this is a deliberate earnings play — note it explicitly)
- [ ] If an active EOD signal applies to this ticker/sector and returns "rejection", discard

**Pass 2 — Cumulative portfolio checks** (applied to the surviving set as a whole):

- [ ] Post-trade cash ≥ 25% of equity — computed as `cash - sum(all surviving candidate costs)`. If the full set would breach, drop the lowest-conviction survivor and re-check. Repeat until set passes.
- [ ] Count of NEW positions (authority `a`) opened today ≤ 3 — if surviving set has more, keep the top 3 by conviction rank.
- [ ] Post-trade invested % ≤ 75% (1% safety margin below the 74% hard cap implied by 25% cash floor) — same drop-lowest-and-retest rule applies.

Any Pass 1 failure → discard candidate. Any Pass 2 failure → iteratively drop lowest-conviction candidates until the set passes.

Log every rejection — individual check failures AND cumulative trims — in the email's "considered but rejected" section with specific reasons.

## Step 8 — Decision

- **Zero candidates survive filter:** STAND DOWN. Build the full email (market context + portfolio snapshot + rejected candidates + stand-down reason).
- **≥1 candidate survives:** rank by conviction (signal strength + portfolio fit). Take top 3 max. Build rationale for each using the template in Step 9.

Conviction ranking heuristic (when tie-breaking):
1. Prefer (c) trim-concentrated actions (risk management first)
2. Then (a) new positions that fill a sector gap with a fresh catalyst
3. Then (b) top-ups on existing winners
4. Then (d) pure rebalance moves

## Step 9 — Rationale Template (~100 words per trade)

Use this exact block structure for each proposed trade. **Word budget: 90-110 words** for the prose portions combined (the 🎯/💡/✅/📏/🛑/🚫/🔬 lines). Hard ceiling 110. If a draft runs over, trim the ✅ bullets first, then the 🚫 list. The ticker/qty/limit header line is not counted in the budget.

Define any jargon on first use (Dylan is learning). Re-use jargon without redefining within the same email — check Appendix C glossary if unsure.

```
═══ TRADE #X OF Y — [AUTHORITY: a/b/c/d] ═══
Ticker: XYZ  |  Side: Buy/Sell  |  Qty: N  |  Limit: $XX.XX  |  Est. cost: $YYY

🎯 SIGNAL: [one sentence — what in the market triggered this]

💡 THESIS: [one sentence in plain English — why the signal matters and why this trade captures it]

✅ WHY THIS TRADE:
  • [portfolio gap filled / sector fit / catalyst alignment]
  • [conviction signal — technical, fundamental, or rotation-based]

🔬 SIGNAL CONFIRMATION: [If an active EOD signal applies, one line: "[SIGNAL_ID] ([Signal Name]): confirms/inconclusive — [why]". Omit entire line if no active signal applies.]

📏 SIZE: $X (Y% of equity, cap $5,091) — [one line on why this size]

🛑 STOP: Z% trailing ([risk tier]) — worst-case loss if stop fires immediately: ~$M

🚫 DISQUALIFIERS: [2-3 conditions that would make this wrong]
```

### Example (for reference — do not reproduce verbatim)

```
═══ TRADE #1 OF 2 — [AUTHORITY: a] ═══
Ticker: XLU  |  Side: Buy  |  Qty: 60  |  Limit: $82.50  |  Est. cost: $4,950

🎯 SIGNAL: 10-year Treasury yield dropped 12bp overnight on soft CPI print (2.8% vs 3.0% expected).

💡 THESIS: When interest rates fall, utility dividends look more attractive vs bonds — institutional money rotates into utilities. XLU is the sector ETF that captures that rotation without single-name risk.

✅ WHY THIS TRADE:
  • Portfolio has zero utility exposure right now
  • XLU based for 2 weeks, today's rate move is a fresh catalyst

📏 SIZE: $4,950 (4.9% of equity) — sized at the 5% cap to test thesis with material exposure.

🛑 STOP: 3% trailing (stable/defensive tier) — worst-case loss if stop fires: ~$149

🚫 DISQUALIFIERS: CPI came in hot; yield rose instead; XLU already up >5% on the week.
```

## Step 10 — Email Assembly

**Subject format:**
- If trades proposed: `Morning Trading Plan — [Date Long] — [N] trades proposed`
- If degraded mode (Phase D): `Morning Brief — [Date Long] — DEGRADED ([failure_class], fallback values from yesterday's EOD + premarket quotes)`
- If standing down: `Morning Brief — [Date Long] — Standing down ([brief reason])`

**Mode header line (use one):**
- Normal: `Mode: DRY-RUN`
- Degraded: `Mode: DRY-RUN-DEGRADED | Source: yesterday's EOD + premarket quotes (Alpaca offline) | Action: VERIFY before next run — see Connectivity Status below`

**Connectivity Status block (include only when degraded):**
Insert this block immediately after the mode header line:
```
🔌 CONNECTIVITY STATUS
Alpaca: OFFLINE | Failure class: [failure_class] | Discovery attempts: [N]/7 | Render status: [up/down/unprobed]
- If render_status = up: Runtime/connector issue, NOT Render. Reconnect at https://claude.ai/settings/connectors
- If render_status = down: Render endpoint unreachable. Check dashboard for alpaca-mcp-server-8zw5
- If render_status = unprobed: WebFetch unavailable; could not distinguish runtime vs Render
Position values below are approximate (qty × premarket_quote). Cash/buying-power frozen at [EOD date].
No new-position proposals in degraded mode — Steps 6-7 skipped.
```

**Body structure (exact order):**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
MORNING TRADING PLAN — [Date]
Mode: DRY-RUN (no orders placed)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📊 ACCOUNT
Equity: $X | Cash: $Y (Z%) | Invested: $A (B%)
Day change: $C (D%) | Since inception: $E (F%)
Buying power: $G

🔔 OVERNIGHT EVENTS
- [Any stop-outs detected in Step 2, with ticker/qty/exit price/approx P/L. If none, write "None."]

🧭 ACTIVE SIGNALS (from EOD research)
[For each active signal from Step 1.5, render TWO lines per signal — a header and the full rule. The rule is rendered verbatim (not summarized) so midday can mechanically evaluate it. Format:

- [SIGNAL_ID] — [SIGNAL_NAME]
  Rule: [full CHECK_RULE text, verbatim from EOD block]

If none: write a single line "No fresh EOD signal research available" or "No promoted signals active this week."]

Signals pending review:
- [Any NEEDS_REVIEW signals, format: `[SIGNAL_ID] — [SIGNAL_NAME] — [FINDING_SUMMARY]`. Omit section if none.]

🌍 MARKET CONTEXT
- [Signal 1]
- [Signal 2]
- [Signal 3]

Sector rotation (1-day):
[Render each sector as: `{emoji} {TICKER}  {bars}{padding} {sign}{abs%}%`
- emoji: 🟢 if pct_change > +0.5%, 🔴 if < -0.5%, ⚪ otherwise
- bars: full block "█" repeated round(abs(pct_change) / 0.3) times, MAX 10 (one block per 0.3%, capped at 3%)
- padding: spaces to right-pad bars to 10 chars
- Sort best-to-worst (top performer first)
- Example:
  🟢 XLK  ██████████ +1.58%
  🟢 XLE  ███        +0.82%
  ⚪ XLB  ▏          +0.17%
  🔴 XLF  ▏          -0.24%
  🔴 XLRE ██         -0.67%]

Interpretation: [one line reading of the pair analysis]

📁 PORTFOLIO SNAPSHOT

| Ticker | Qty | Avg Cost | Current | Unrealized P/L | % Equity | Stop Dist | 🔔 |
|--------|-----|----------|---------|----------------|----------|-----------|-----|
| ...    |     |          |         |                |          |           | EARN if within 5d |

Position P/L (visual):
[For each position, render: `{emoji} {TICKER}  {bars}{padding} {sign}{abs%}%`
- emoji: 🟢 if unrealized_plpc > 0, 🔴 if < 0, ⚪ if exactly 0
- bars: full block "█" repeated round(abs(unrealized_plpc * 100) / 1) times, MAX 10
- padding: spaces to right-pad bars to 10 chars
- Sort by abs(unrealized_plpc) descending]

Stop headroom (visual):
[For each position, render: `{TICKER}  [{filled}{empty}] {headroom}% to stop {warn}`
- filled: full block "█" × round(headroom_pct / 0.5), MAX 10
- empty: "░" filling rest to 10 chars
- warn: "⚠" if headroom < 2%, else ""
- Sort by headroom ascending — riskiest first]

Flags:
- [At-risk positions, if any]
- [Concentrations >12%, if any]
- [Upcoming earnings within 5 days, if any — reference the 🔔 column]

💼 TRADES PROPOSED ([N] total)
[Rationale block 1 per Step 9]
[Rationale block 2]
[...]

🗑️ CONSIDERED BUT REJECTED
- TICKER: [one-line reason, referencing the specific failing guardrail]
- TICKER: [...]

📈 PORTFOLIO IMPACT (if all proposed trades were executed)
Cash: $X → $Y (Z% → W%)
Invested: $A → $B (C% → D%)
Position count: N → M
Sector mix changes: [+XLU, -FCX partial, etc.]

🛑 OPEN TRAIL STOPS (status)

| Ticker | Qty | Trail % | Stop | HWM | Headroom |
|--------|-----|---------|------|-----|----------|
| ...    |     |         |      |     |          |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⚠️ DRY-RUN REMINDER
No orders were placed by this agent. All trades above are proposals for review.
To execute any of these, reply to this thread or open an interactive Claude session.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### Standdown email format (when Step 8 says zero candidates)

```
Subject: Morning Brief — [Date] — Standing down ([brief reason])

━━━━━━━━━━━━━━━━━━━━━━━━━━━━
MORNING BRIEF — [Date]
Mode: DRY-RUN — STANDING DOWN
━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Reason for standdown: [specific — e.g., "all 4 candidates failed guardrails: 2 over size cap, 1 on earnings proximity, 1 already had working buy order"]

📊 ACCOUNT
[same as normal email]

🌍 MARKET CONTEXT
[same]

📁 PORTFOLIO SNAPSHOT
[same]

🗑️ CONSIDERED BUT REJECTED
[all candidates with their reasons — this is especially valuable on standdown days]

🛑 OPEN TRAIL STOPS
[same]

Next scheduled run: Midday Portfolio Check at 12:00 PM ET.
```

## Step 11 — Create Gmail Draft (dual-format)

### Construct HTML body

Build an HTML version of the email body that mirrors the plaintext sections from Step 10 with proper visual styling. Use the visual conventions in Appendix D. The HTML body should include:
- Section headers as styled `<h2>` (per Appendix D)
- Account summary as a styled box
- Portfolio snapshot as a real `<table>` with color-coded P/L cells
- Position P/L visual rendered with HTML/CSS bars (per Appendix D)
- Sector rotation as styled bars (use the same horizontal-bar pattern, color-coded by sign)
- Stop headroom gauges as HTML bars (red <2%, amber 2-3%, green >3%)
- Trade rationale blocks as styled "cards" with 🎯/💡/✅/🔬/📏/🛑/🚫 sections clearly delineated
- Same emoji section anchors as plaintext

### Draft creation

Call Gmail `create_draft` with:
- **To:** `{{RECIPIENT_EMAIL}}`
- **Subject:** per Step 10
- **body:** the plaintext body per Step 10. **MUST remain plain text** — midday parses this via `plaintextBody`. Preserve the emoji markers (📊 🌍 📁 🔔 🧭 💼 🗑️ 🛑) — midday uses them as section anchors.
- **htmlBody:** the styled HTML body constructed above. This is what Dylan sees rendered in Gmail.

Both fields are required.

### Failure handling for create_draft

If `create_draft` returns an error specifically related to the htmlBody:
1. Retry once WITHOUT the htmlBody parameter, sending only the plaintext `body`.
2. Note in the trigger run log: "HTML body rejected; sent plaintext only."
3. Do not let HTML failure block the report.

If Gmail MCP fails entirely: log the full plaintext body to the trigger's run output so Dylan can find it in the Anthropic trigger logs at claude.ai/code/scheduled.

### Post-create verification (catch silent connector failures)

The Gmail connector has been observed returning a fake-success response from `create_draft` without persisting the draft (incident 2026-05-01 — all three agents reported "Gmail draft created successfully" but no drafts landed in Gmail; suspected root cause was OAuth invalidation following a Google password rotation, leaving the connector returning success silently). Verify before exiting:

1. Capture the `draftId` returned by `create_draft` and the exact `Subject` string you used.
2. Sleep 5 seconds — Gmail's draft indexing has minor lag.
3. Call `list_drafts` with `query: "subject:\"<exact subject>\""`, `pageSize: 5`. Treat as a match only if a returned draft's `date` is within the last 5 minutes.
4. **≥1 matching recent draft:** verification passes. Proceed to Step 12.
5. **Zero matches:** the connector silently failed. Retry `create_draft` once with identical parameters. Sleep 5s. Re-run the `list_drafts` probe.
6. **Still zero matches after retry:** output to trigger run log:
   ```
   DRAFT_VERIFY_FAILED | subject: <subject> | claimed_id_1: <id1> | claimed_id_2: <id2 or "n/a"> | Gmail connector likely broken — reconnect at https://claude.ai/settings/connectors before next run
   ```
   Then EXIT WITH AN ERROR (non-zero). The trigger schedule UI must show RED for this run — green-with-no-draft is the exact failure mode this step exists to surface.

## Step 12 — Completion

Output a single status line:
```
Morning brief drafted | Trades proposed: [N] | Equity: $X | Mode: DRY-RUN | Status: OK
```

Exit.

---

## Appendix A — Standdown vs Degraded vs Trades-Proposed (decision tree)

Three terminal states are NOT the same — pick the right one:

**Trades proposed** (the common active-day path):
- Step 1 succeeded, ≥1 candidate passed Step 7's guardrails.
- Subject: `Morning Trading Plan — [Date] — [N] trades proposed`.

**Monitoring / no candidates** (also common):
- Step 1 succeeded but Step 8 produced zero survivors.
- Subject: `Morning Brief — [Date] — Standing down (zero candidates passed guardrails)`.
- Body includes the rejected-candidates section so the standdown is auditable.

**Degraded mode** (Phase D fallback, post-2026-05-10):
- Trigger: Step 1 Phase A or B failed (discovery exhausted, Render down, or Alpaca API errored), but yesterday's EOD email + premarket quotes were retrievable.
- Subject: `Morning Brief — [Date] — DEGRADED (...)`.
- Body includes Connectivity Status block + approximate position values + signals carried from yesterday's EOD.
- No new-position proposals (degraded data, unreachable order plumbing).
- Dylan reads the Connectivity Status block and acts on it before the next scheduled run.

**True standdown** (last resort, no portfolio visibility possible):
1. Kill switch is active (Step 0).
2. Step 1 Phase C account check failed (account blocked / suspended).
3. Market closed all day — holiday (Step 1 Phase C).
4. Step 1 Phase D also failed (no recent EOD email AND no Yahoo/financial-datasets quotes available).
5. Unrecoverable error at any step — log the error in the draft body.

Pre-2026-05-10 the agent treated discovery failures as standdowns. Post-2026-05-10 those become degraded mode. The May 8 midday standdown pattern ("STANDING DOWN — Alpaca tools never registered after 5 discovery attempts") would now be a DEGRADED report at any agent.

## Appendix B — Safety Rails

- **No pre-market trading.** Trigger fires at 9:00 AM ET, market opens at 9:30. Use `time_in_force: day` on proposed limits — they'll sit until open. Never use extended hours.
- **No options, crypto, or shorts** in this playbook. Those require separate playbooks.
- **No leverage beyond existing 2x margin.** If a proposed trade would require more than non-marginable buying power × 2, discard.
- **No position in a ticker that stopped out in the last 5 trading days** (avoid chasing a just-broken thesis).
- **Flag earnings within 5 trading days** on any ticker being proposed. Prefer to avoid the name or note the earnings risk explicitly in the rationale.
- **Position sizing math shown explicitly** in every rationale block — Dylan is learning.

## Appendix B2 — Execute-mode client_order_id contract (for future morning-execute-v1)

When this playbook migrates to execute mode (no longer dry-run), every `place_stock_order` / `place_option_order` / `place_crypto_order` call MUST set `client_order_id` using this canonical format:

```
morning-{YYYY-MM-DD}-{ticker}-{authority}-{seq}
```

- `{YYYY-MM-DD}` = ET trading date
- `{ticker}` = symbol
- `{authority}` ∈ {`open`, `topup`, `trim`, `stop`} — matches Step 6 authority labels
- `{seq}` = 3-digit zero-padded sequence within morning + date (start at 001)

Example: `morning-2026-04-23-XLK-open-001` for morning's first new-position on XLK on Apr 23.

When placing the corresponding trail stop for a new position, use `authority = stop`: `morning-2026-04-23-XLK-stop-001`.

EOD reconciliation depends on this format. Trail stops placed with the legacy `protect-*` prefix (from earlier sessions) continue to work — EOD attributes them as mechanical exits.

## Appendix D — HTML Visual Conventions (for htmlBody)

Use these consistently. Inline styles only (Gmail strips `<style>` blocks).

### Color tokens
- Positive: `color: #16a34a` (green) | Negative: `color: #dc2626` (red) | Neutral: `color: #6b7280` (gray) | Warning: `color: #eab308` (amber)
- Section header dark: `color: #1e293b` | Accent blue: `#3b82f6`

### Section header
```html
<h2 style="font-family:Arial,sans-serif;color:#1e293b;border-bottom:2px solid #3b82f6;padding-bottom:4px;margin-top:24px;">📊 Section Title</h2>
```

### Account summary box
```html
<div style="background:#f8fafc;border:1px solid #e2e8f0;border-radius:8px;padding:16px;margin:12px 0;font-family:Arial,sans-serif;">
  <div style="font-size:24px;font-weight:bold;">$101,500</div>
  <div style="color:#16a34a;font-size:16px;">Day: +$640 (+0.60%)</div>
  <div style="color:#6b7280;margin-top:4px;">Cash: $X (Y%) | Invested: $A (B%) | Buying Power: $G</div>
</div>
```

### Standard table (portfolio, trail stops, reconciliation)
```html
<table style="width:100%;border-collapse:collapse;font-family:Arial,sans-serif;font-size:14px;margin:8px 0;">
  <tr style="background:#1e293b;color:white;">
    <th style="padding:8px;text-align:left;">Ticker</th>
    <th style="padding:8px;text-align:right;">Qty</th>
    <th style="padding:8px;text-align:right;">P/L</th>
  </tr>
  <!-- alternate row backgrounds: #ffffff and #f8fafc -->
  <tr style="background:#ffffff;">
    <td style="padding:8px;">SPY</td>
    <td style="padding:8px;text-align:right;">22</td>
    <td style="padding:8px;text-align:right;color:#16a34a;">+$612</td>
  </tr>
</table>
```

### Horizontal bar (P/L, sector rotation, stop headroom)
Width = `min(abs(value) * scale, 100)` percent. Color = green if positive, red if negative, color-graded for stop headroom (red <2%, amber 2-3%, green >3%).
```html
<div style="display:flex;align-items:center;gap:8px;font-family:Arial,sans-serif;font-size:14px;margin:2px 0;">
  <span style="display:inline-block;width:60px;font-weight:bold;">NVDA</span>
  <div style="flex:1;background:#f1f5f9;height:14px;border-radius:2px;overflow:hidden;max-width:300px;">
    <div style="width:[N]%;height:100%;background:#16a34a;"></div>
  </div>
  <span style="color:#16a34a;font-weight:bold;width:70px;text-align:right;">+10.4%</span>
</div>
```

### Trade rationale card
```html
<div style="background:#fefce8;border-left:4px solid #eab308;border-radius:4px;padding:12px 16px;margin:12px 0;font-family:Arial,sans-serif;">
  <div style="font-weight:bold;font-size:15px;color:#1e293b;">═══ TRADE #1 OF 3 — [AUTHORITY: c] ═══</div>
  <div style="margin-top:4px;">Ticker: <strong>SPY</strong> | Side: <strong>Sell</strong> | Qty: 7 | Limit: $708 | Est. proceeds: $4,956</div>
  <div style="margin-top:8px;"><strong>🎯 SIGNAL:</strong> [text]</div>
  <div><strong>💡 THESIS:</strong> [text]</div>
  <div><strong>✅ WHY:</strong> [text]</div>
  <div><strong>🔬 SIGNAL CONFIRMATION:</strong> [text or omit]</div>
  <div><strong>📏 SIZE:</strong> [text]</div>
  <div><strong>🛑 STOP:</strong> [text]</div>
  <div><strong>🚫 DISQUALIFIERS:</strong> [text]</div>
</div>
```

### Body wrapper
```html
<body style="font-family:Arial,sans-serif;max-width:720px;margin:0 auto;padding:16px;">
  <!-- all sections -->
</body>
```

### Rules
- Inline styles only — no `<style>` blocks, no JS, no external CSS
- No external images (charts deferred)
- Mirror plaintext section order exactly so the formats are comparable
- Preserve emoji — Gmail renders natively

## Appendix C — Jargon Glossary (define on first use in email)

When these terms appear in an email, add a one-line definition the first time they appear on any given day:

- **Bp / basis points** — 1bp = 0.01%. "Yield dropped 12bp" = yield fell by 0.12 percentage points.
- **Trail stop / trailing stop** — an automatic sell order that follows the stock's high-water mark down by a fixed percentage.
- **HWM** — high-water mark. The highest price the stock has hit since the trail stop was placed.
- **Risk-on / risk-off** — risk-on days see investors buy growth/cyclical assets; risk-off sees them rotate to cash, bonds, gold, or defensives.
- **Sector rotation** — institutional capital moving between sectors based on the economic cycle, rates, or sentiment.
- **Dr. Copper** — old Wall Street nickname for copper, because copper demand leads the economic cycle (construction, electronics, wiring).
- **Extended / overextended** — a stock or index that has rallied sharply without pause, increasing the chance of a pullback.

---

*End of playbook. The agent should exit after Step 12.*
