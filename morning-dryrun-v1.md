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

## Step 0 — Kill Switch Check

Search Gmail for `label:TRADING-PAUSED newer_than:2d`.

If ANY match found:
- Do not proceed with any research or trading logic.
- Create a Gmail draft titled `Morning Brief — [Date] — STANDING DOWN (kill switch active)`.
- Body: one line stating "Kill switch detected. Remove the `TRADING-PAUSED` label to resume."
- Exit.

## Step 1 — Health Check (cold-start tolerant)

The Alpaca MCP is hosted on Render's free tier, which spins down after 15 min idle. The morning trigger fires after ~17 hours of overnight idle, so the first call may need 30-60 seconds for cold start. Use this retry pattern:

1. **Attempt 1:** Call Alpaca `get_clock`. If it succeeds in <15s, proceed to step 5.
2. **Attempt 2:** If attempt 1 failed or timed out, wait 30 seconds and retry `get_clock`. If success, proceed.
3. **Attempt 3:** If attempt 2 failed, wait 60 more seconds and retry `get_clock` one final time. If success, proceed.
4. **All 3 failed:** standdown email with subject `Morning Brief — [Date] — STANDING DOWN (Alpaca offline after 3 attempts, ~105s budget exhausted — likely Render service down or extended cold start)`. Exit.
5. Call Alpaca `get_account_info`. Verify:
   - `status == "ACTIVE"`
   - `trading_blocked == false`
   - `account_blocked == false`
   - `trade_suspended_by_user == false`
6. If any fails: standdown email describing which check failed. Exit.
7. Call `get_calendar` for today. If market is closed the whole day (holiday), standdown email with subject `Morning Brief — [Date] — Market closed`.
8. Record baseline metrics: `equity`, `cash`, `last_equity`, `portfolio_value`, `buying_power`, `non_marginable_buying_power`.

## Step 1.5 — Active Signal Watchlist (from EOD research)

The EOD agent runs structured signal research each weekday and embeds a machine-readable block at the bottom of its email. Load the most recent one so morning decisions benefit from last night's research.

1. Search Gmail: `from:{{RECIPIENT_EMAIL}} subject:"EOD Report" newer_than:3d` — take the most recent matching message. Call `get_thread` with `messageFormat: FULL_CONTENT` to retrieve the body (snippets are truncated and will miss the SIGNAL_RESULT block).
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
- **Earnings proximity**: for each holding, call `mcp__yahoo-finance__get_stock_earnings` — flag any position with earnings within 5 trading days with `🔔 EARN`
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

Fetch news using this **explicit fallback chain** (try in order, stop when 3 usable headlines are found):
1. `mcp__tradingview__financial_news` → if returns 0 items or errors, continue
2. `mcp__yahoo-finance__get_stock_news` for SPY → top 3
3. `mcp__yahoo-finance__get_stock_news` for QQQ → top 3 if SPY feed was empty
4. If all three produce nothing, record `news_feed = empty` and note in the email: "No fresh market news retrievable — tape read is price-only."

Derive 3–5 plain-English signals. Examples of what "a signal" looks like:
- "10yr yield -12bp overnight on soft CPI print (2.8% vs 3.0% est)"
- "NVDA +2% pre-market after semi earnings beat"
- "Gold +0.8% on a risk-on day — institutions hedging"
- "Dollar breakout to 6-month highs — headwind for multinationals"

If nothing material is happening, note that explicitly: "quiet tape, no fresh catalysts."

## Step 4 — Position-Level Scan

For each existing holding, check:
- Latest quote vs. trail stop → headroom %
- News in last 24h via `mcp__yahoo-finance__get_stock_news`
- Earnings within 5 trading days via `mcp__yahoo-finance__get_stock_earnings` — flag this name for tighter risk management or avoidance of top-ups (result may already be cached from Step 2's earnings-proximity sweep — reuse if so)

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
- If standing down: `Morning Brief — [Date Long] — Standing down ([brief reason])`

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

Sector rotation (1-day): [ranked list, e.g., "XLK +1.58% > XLE +0.82% > ... > XLRE -0.67%"]
Interpretation: [one line reading of the pair analysis]

📁 PORTFOLIO SNAPSHOT

| Ticker | Qty | Avg Cost | Current | Unrealized P/L | % Equity | Stop Dist | 🔔 |
|--------|-----|----------|---------|----------------|----------|-----------|-----|
| ...    |     |          |         |                |          |           | EARN if within 5d |

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

## Step 11 — Create Gmail Draft

Call Gmail `create_draft` with:
- **To:** `{{RECIPIENT_EMAIL}}`
- **Subject:** per Step 10
- **Body:** per Step 10 — **MUST be plain text**, not HTML. The midday agent parses this body via `plaintextBody` and HTML-only bodies will be unparseable. Preserve the emoji markers (📊 🌍 📁 🔔 🧭 💼 🗑️ 🛑) — midday uses them as section anchors.

If Gmail MCP fails: log the full email body to the trigger's run output so Dylan can find it in the Anthropic trigger logs at claude.ai/code/scheduled.

## Step 12 — Completion

Output a single status line:
```
Morning brief drafted | Trades proposed: [N] | Equity: $X | Mode: DRY-RUN | Status: OK
```

Exit.

---

## Appendix A — Standdown Scenarios

Create a standdown draft (not a trade draft) when any of:
1. Kill switch is active (Step 0)
2. Alpaca unreachable after retry (Step 1)
3. Account not ACTIVE or blocked (Step 1)
4. Market closed all day — holiday (Step 1)
5. Zero candidates pass guardrails (Step 8)
6. Unrecoverable error at any step — log the error in the draft body

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
