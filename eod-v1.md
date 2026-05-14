# End-of-Day Reporting & Research Playbook — v1

*Paste this into `github.com/Dylanxmt/finance-playbooks/eod-v1.md`. Version: 2026-04-22 (r2, post-simulation patches).*

*This file is fetched at runtime by the Anthropic scheduled trigger. The EOD agent is read-only — it never proposes or executes trades. Its job is to (1) produce an authoritative day report that the next morning agent can parse, and (2) run a best-effort signal research investigation.*

---

## 🔴 STRICT READ-ONLY MODE

1. You MUST NOT call any order-execution tools: `place_stock_order`, `place_crypto_order`, `place_option_order`, `cancel_order_by_id`, `replace_order_by_id`, `close_position`, `close_all_positions`, `exercise_options_position`.
2. This is end-of-day reporting + research only. No trades, no proposals, no orders.
3. Read-only Alpaca tools are fine: all `get_*` tools.
4. WebSearch/WebFetch are fine for Step 6 signal research.

---

## Architecture (why this playbook looks this way)

EOD has TWO jobs:
1. **Report** — produce the authoritative day snapshot. Morning's parsing depends on this. **Mandatory.**
2. **Research** — run one signal-backlog hypothesis to completion, grade it, emit structured result. **Best-effort.**

If research errors or times out, skip to email assembly and ship the report. Morning handles a missing signal block gracefully.

---

## Context injected by trigger

- **Account:** Alpaca paper `{{ACCOUNT_ID}}` (provided in trigger context)
- **Recipient:** `{{RECIPIENT_EMAIL}}` (provided in trigger context)
- **Today:** the date the trigger fires
- **Signal target:** injected from trigger prompt (see Appendix B for current rotation)
- **Research time budget:** 8 minutes for Step 6 (hard cap — after that, emit "research skipped" note and move on)

## Kill switch note

EOD does NOT check either kill path (repo file `kill-switch/state.txt` OR Gmail `TRADING-PAUSED` label). Both paths exist to pause TRADING; EOD only reports. Reports are always useful — especially during incidents when Dylan most needs visibility. Run every day regardless of kill switch state.

If you want EOD to also pause during a kill (e.g., suspected agent compromise), pause the trigger directly in claude.ai/code/scheduled UI. That kills all activity from this agent, not just trading.

---

## Step 1 — Health Check (discovery-tolerant + degraded-mode fallback)

The Alpaca MCP connector lives behind two layers: (a) the Render-hosted MCP server itself, and (b) the Anthropic agent runtime's discovery / tool-registration layer. UptimeRobot keeps the Render dyno warm (verified 2026-04-27 — initialize handshake responds in ~140ms), so the dominant failure mode is **runtime discovery taking longer than the first attempt**, not Render cold-start.

**EOD has always been degradation-tolerant** (Appendix A: "EOD ships the report even when individual steps fail"). What changes 2026-05-10 is the FAILURE CLASSIFICATION and the FALLBACK DATA SOURCE — instead of just shipping an Alpaca-less email, EOD now distinguishes "runtime slow" from "Render down" via probing, uses exponential backoff for cleaner retry semantics, and pulls today's morning email + Yahoo current quotes for a richer degraded report.

Failures at any phase below set `alpaca_available = false` and skip live-data steps. EOD still ships the report — that contract is unchanged. The patch sharpens the diagnostics that go into the email body.

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
4. Discovery budget: 45s priming + ~250s retry = ~5 min worst case.

**Render probe (sub-step, runs once after attempt 3 fails):**
Use `WebFetch` on `https://alpaca-mcp-server-8zw5.onrender.com/` with a 15s timeout. Interpret:
- Any HTTP response (200, 404, 405, 400, 500, etc.) → set `render_status = "up"`. The server is live; runtime hasn't finished tool negotiation. KEEP retrying through attempt 7.
- Connection refused, DNS error, timeout, or 502/503/504 → set `render_status = "down"`. Stop retrying — set `alpaca_available = false`, `failure_class = "render_down"`. Skip remaining attempts. Go to Phase D.
- If `WebFetch` is unavailable in this runtime, set `render_status = "unprobed"` and continue retries on the schedule above.

Capture and persist for the email body: `discovery_attempts_made` (1-7), `render_status`, `failure_class`.

**Phase B — probe Alpaca.**
5. Call `get_clock` using the `alpaca_tool_prefix` captured in Phase A (i.e., `{alpaca_tool_prefix}__get_clock`). Allow up to 30s.
6. If it fails or times out, wait 30s and retry once.
7. If both attempts fail *despite tools being registered*: set `alpaca_available = false`, `failure_class = "alpaca_api_error"` (include error text). Go to Phase D.

**Phase C — verify account.**
8. If `alpaca_available`: call `get_account_info`. Record baseline: `equity`, `cash`, `last_equity`, `portfolio_value`, `buying_power`. Set `position_source = "alpaca-live"`.
9. Record market state — was the market open today? If holiday / closed, note it but still produce a minimal report.
10. Proceed to Step 2.

**Phase D — degraded-mode fallback.**
Entered only when `alpaca_available = false` from Phase A or B. Phase D produces a degraded EOD report using today's morning email's portfolio snapshot + current quotes. Still ships — true standdown only when fallback also fails.

11. Search Gmail: `in:draft from:{{RECIPIENT_EMAIL}} (subject:"Morning Trading Plan" OR subject:"Morning Brief") newer_than:1d`. Take today's match. **The `in:draft` flag is load-bearing** — morning reports are saved as drafts and `search_threads` excludes drafts by default.
12. Call `get_thread` with `messageFormat: FULL_CONTENT`. Parse the morning body for:
    - Portfolio table rows: ticker, qty, avg cost, opening price.
    - Account summary: morning's `equity`, `cash`, `buying_power` (label "as of morning, [time]").
    - The 9-AM stop-headroom snapshot if rendered (used to estimate which positions were close to stops at open — note these may have stopped out intraday and we have no way to verify).
13. For each ticker, fetch a current closing-area quote:
    - First try `mcp__yahoo-finance__get_stock_price`.
    - If that fails, try `mcp__financial-datasets__get_current_stock_price`.
    - If both fail, mark row `price_unavailable`.
14. Compute approximate per-position values:
    - `position_value ≈ qty × current_price`
    - `unrealized_pl ≈ (current_price − avg_cost) × qty`
    - `day_change_pct ≈ (current_price / morning_open − 1) × 100`
15. Compute approximate equity: `approximate_equity ≈ sum(position_values) + cash_from_morning`. Mark `†` and footnote "approximate — Alpaca offline, computed from this morning's snapshot + current quotes. Cannot detect intraday fills, stop-outs, or cash changes."
16. Set `position_source = "fallback (today's morning email + current quotes)"`. **DEGRADED-MODE CONSTRAINTS for EOD:**
    - **Skip Step 4 (Day Reconciliation) entirely** — no fills data without Alpaca.
    - **Skip Step 5 (Post-Mortems) entirely** — no stop-out detection without Alpaca.
    - Step 6 (Signal Research) STILL RUNS — web-only, independent of Alpaca. EOD's research deliverable remains intact.
    - Step 7 (email assembly) renders DEGRADED format with Connectivity Status block and the approximate snapshot.
    - Morning agent next day will pull DEGRADED EOD's signal block normally; the portfolio snapshot it uses will be approximate. Note this clearly in the email so morning's degraded-mode fallback (which keys off "EOD Report" emails) can detect it via the DEGRADED tag in the subject.
17. If today's morning email is also missing OR all current-price lookups fail: this is a **true standdown**. Subject: `EOD Report — [Date] — STANDING DOWN (degraded fallback failed: [reason])`. Body explains what was tried. Exit.

**Total budget:** ~5 min Phase A + ~60s Phase B + ~30s Phase C + ~90s Phase D = ~8 min. Agent fires 3:55 PM ET — well within budget regardless of path.

## Step 2 — Parse Today's Morning & Midday Emails

Pull the day's narrative from the agents that already ran:

1. **Morning:** search Gmail with
   ```
   in:draft from:{{RECIPIENT_EMAIL}} (subject:"Morning Trading Plan" OR subject:"Morning Brief") newer_than:1d
   ```
   Take the most recent. Call `get_thread` with `messageFormat: FULL_CONTENT`. **The `in:draft` flag is required** — morning reports persist as drafts; Gmail's `search_threads` excludes drafts by default. Extract from `plaintextBody`:
   - Proposed tickers + authority
   - Considered-but-rejected tickers
   - Flagged positions
   - Overnight events
   - **If `plaintextBody` is empty or contains only a shell string like "see HTML version":** set `morning_parse_status = unparseable`. Do NOT fail. Fall back to the `client_order_id` prefix convention (Step 4) for reconciliation.
2. **Midday:** search Gmail with
   ```
   in:draft from:{{RECIPIENT_EMAIL}} subject:"Midday Brief" newer_than:1d
   ```
   Take the most recent. Extract (same way):
   - Reactive trades proposed (if any)
   - Intraday stop-outs noted
   - Tape-read label (Confirms / Neutral / Flips)
   - Same `plaintextBody` fallback rule as morning.
3. Hold both as `morning_brief` and `midday_brief` records for Step 4 reconciliation.
4. If either email is missing entirely (not just unparseable), record the gap — don't fail. Morning/midday could have errored, been killed, or simply not deployed yet (dry-run phase).

## Step 3 — Portfolio Snapshot (EOD)

Gather in this order — **cache `get_all_positions` FIRST** so that Step 5 post-mortems can reference pre-stop-out avg_cost for names that have already exited (they won't appear in a later call):

1. `get_all_positions` → full list. **Snapshot this at the start** and hold as `positions_snapshot_eod` for later steps.
2. `get_orders` with `status: open` → current trail stops + any still-working limits
3. `get_account_activities_by_type` with `activity_type: FILL`, `date: <today YYYY-MM-DD>` → authoritative fill log
4. For each stopped-out ticker (see below), call `get_account_activities_by_type` again with `activity_type: FILL`, `until: <today 00:00 ET>`, plus a large enough `after` window (e.g., 45 days back) to find the ORIGINAL buy. Keep the earliest BUY fill per symbol — Step 5 needs it.
5. `get_portfolio_history` with `period: 1M, timeframe: 1D` → last 30 days equity curve

### Handling the fills response

`get_account_activities_by_type` returns ONE entry per partial fill, plus one final `fill` entry per order. For today's orders, **group by `order_id` and take the entry where `leaves_qty == 0`** (or the one with `order_status: filled`) as the authoritative logical fill. Sum `cum_qty` and use the final `price` (all partials fill at the same limit so price is uniform in practice, but defensive math = `sum(partial_qty × partial_price) / sum(partial_qty)`).

Discard all intermediate `partial_fill` entries from the email table — they are audit data, not user-facing.

### Compute

- **Fills today (logical)** — post-dedup list: time (final fill), ticker, side, total_qty, price, `client_order_id`, attribution (see Step 4 prefix convention)
- **Stop-outs today** — subset of logical fills where the underlying order's `order_type == trailing_stop` and `side == sell`. Determine `order_type` by cross-referencing `order_id` to today's `get_orders` closed list.
- **Realized P/L today** — for each sell fill: `(fill_price - avg_cost_pre_sell) × qty`. Pull `avg_cost_pre_sell` from `positions_snapshot_eod` if the name is still held; else from the original BUY fill's price (Step 3 item 4).
- **Unrealized P/L (current positions)** — sum of `unrealized_pl` across `positions_snapshot_eod`
- **Day P/L** — `equity - last_equity`
- **Since inception** — `equity - 100000`, plus % (**hardcode start date as 2026-04-08**; do NOT trust `base_value_asof` from `get_portfolio_history`, which reports 2026-04-06 and conflicts)
- **Trail stop inventory** — for each position, record trail %, stop price, HWM, headroom % to stop

### Note on get_portfolio_history response shape

The API returns parallel arrays: `{timestamp: [...], equity: [...], profit_loss: [...], base_value, ...}`. Pair `timestamp[i]` with `equity[i]` by index. `timestamp[i]` is Unix seconds UTC — convert to ET date for display.

## Step 4 — Day Reconciliation

Match the morning/midday plans against what actually happened:

For each morning-proposed trade:
- Did a fill matching its ticker + side + approximate qty occur today? → mark `FILLED`
- Is there a working limit order? → mark `WORKING`
- Neither? → mark `NOT_EXECUTED` (Dylan didn't action it manually, or it wasn't placed in dry-run)

For each midday-proposed trade: same reconciliation.

For each intraday stop-out: was it flagged in morning (stops-at-risk) or midday? Track for post-mortem generation in Step 5.

Output: a short table in the email's "📋 DAY RECONCILIATION" section showing proposed vs. actual.

### Client-order-id prefix convention (reconciliation fallback)

When morning/midday body is unparseable (Step 2 `parse_status = unparseable`), reconcile fills via `client_order_id` prefix.

**Canonical format:** `{agent}-{YYYY-MM-DD}-{ticker}-{authority}-{seq}`

- `{agent}` ∈ {`morning`, `midday`, `protect`, `manual`, `rebal`}
- `{YYYY-MM-DD}` is the ET trading date the order was placed
- `{ticker}` is the underlying symbol
- `{authority}` ∈ {`open`, `topup`, `trim`, `tighten`, `stop`} — matches the playbook authority labels (a/b/c/c'/mechanical)
- `{seq}` is a 3-digit zero-padded sequence within agent+date, e.g., `001`, `002`

**Example:** `morning-2026-04-23-XLK-open-001` is morning's first new-position order for XLK on Apr 23.

| Prefix | Source | Expected behavior |
|--------|--------|-------------------|
| `morning-*` | Morning agent (execute mode) | Report as agent fill; match to morning_brief proposed-trades list |
| `midday-*` | Midday agent (execute mode) | Same for midday |
| `protect-*` | Trail stop (legacy or agent-placed) | Attribute as mechanical; goes to stop-out list if filled |
| `rebal-*`, `manual-*`, or no prefix | Dylan interactive session | Report as "Manual rebalance" — not attributable to any scheduled agent |

If morning/midday parsed successfully AND fills have matching prefixes, prefer the email's proposed-trades list as the source of truth. If parse failed, prefixes are the fallback.

### Modification attribution rule

If a `replace_order_by_id` or `cancel_order_by_id` is issued by a DIFFERENT agent than the one that placed the original order (e.g., midday tightens a morning-placed stop), the REPLACEMENT order should be placed with a NEW `client_order_id` reflecting the acting agent. Example:
- Morning places: `morning-2026-04-23-XLK-open-001` (buy limit that later fills)
- Morning places corresponding stop: `morning-2026-04-23-XLK-stop-001`
- Midday tightens that stop via replace: new order gets `midday-2026-04-23-XLK-tighten-001`

EOD attribution now correctly shows midday as the acting agent for the tightening, with a one-line note in reconciliation: "XLK stop originally placed by morning, modified by midday at [time]."

If the acting agent keeps the original `client_order_id` (technically allowed but discouraged), EOD attributes to the original placer — document the edge case but don't treat it as an error.

**Going-live note:** when morning/midday migrate to execute mode, their order-placement calls must set `client_order_id` with the canonical format above. Add this to `morning-execute-v1.md` and `midday-execute-v1.md` as a hard requirement when drafted.

## Step 5 — Post-Mortem Generation

For each stop-out today:

1. Retrieve the position's history:
   - **Entry price & date:** from the earliest BUY fill in the historical `get_account_activities_by_type` call made in Step 3 item 4 (filter by `side: buy` + matching symbol, take the earliest by `transaction_time`).
   - **Peak price (HWM):** from the `hwm` field on the trail-stop order record (found in today's closed orders list where `order_type == trailing_stop` and the symbol matches).
   - **Exit price & date:** from today's final stop-out fill (`price` field).
   - **Days held:** `exit_date - entry_date` in calendar days (ET).
   - **Avg cost:** from `positions_snapshot_eod` if the name had been re-cached just before exit, else weighted average of all historical BUY fills for the symbol.
2. Check morning/midday emails for any flag on this name ("at-risk", adverse news, etc.) — did we have warning?
3. Compute: total P/L = `(exit_price - avg_cost) × qty`, % return = `(exit_price / avg_cost - 1) × 100`, drawdown from peak = `(exit_price / hwm - 1) × 100`.
4. Generate a short post-mortem block for the email:

```
📓 POST-MORTEM — [TICKER]
Entry: $X.XX on [date] | Exit: $Y.YY on [date] (trail stop)
P/L: $Z.ZZ ([+/-]W.W%) | Held: N days | Trail: M%
Peak: $P.PP (drawdown from peak: -D.D%)
Warning signs before stop-out: [yes — with detail / none — stop worked cleanly]
Thesis reflection: [1-2 sentences — did the original thesis hold? what changed?]
Suggested backlog item: [one-line hypothesis, optional — goes to weekly signal review]
```

If no stop-outs today: omit this section entirely.

## Step 6 — Signal Research (Best-Effort)

**Hard time budget: 8 minutes.** If you exceed this, abandon research, set `research_status = skipped`, and proceed to Step 7. Better to ship the report than stall.

**Minimum search count: 3.** Do not short-cut to 1 or 2 searches even if the first result seems definitive — the Performance grade should reflect triangulated evidence, not a single headline. If the 8-minute budget forces stopping before 3 searches complete, set `research_status = skipped` (not "completed with partial data").

1. Read the injected `Signal target` block from the trigger prompt. It contains: hypothesis, channel, target tickers.
2. Run 3-5 web searches (minimum 3):
   - Academic/empirical evidence for this signal type
   - Recent real data for 2-3 of the target tickers
   - Cross-reference: did this signal fire before a notable move in the past 90 days?
3. Grade:
   - **Performance (0-5)**: how actionable is this signal? (0=dead end, 3=usable, 5=exceptional)
   - **Creativity (0-5)**: how original? (0=obvious, 3=creative cross-ref, 5=paradigm insight)
4. Apply decision rule:
   - Performance ≥ 4 → **AUTO-PROMOTE**
   - Performance ≤ 1 AND Creativity ≤ 1 → **AUTO-DROP**
   - Everything else → **NEEDS_REVIEW**
5. If AUTO-PROMOTE: write a `CHECK_RULE` that the morning agent can apply mechanically. The rule must be specific enough that morning can evaluate it without interpretation (e.g., "Before entering any semiconductor stock, search for SEC Form 4 cluster insider buying in last 30 days — 3+ insiders = confirmation, absence = neutral, cluster selling = reject"). Rule may span multiple sentences if thresholds are complex, but must fit on a single line in the emitted block (no newlines within the field value).
6. Assign a stable `SIGNAL_ID`:
   - If the trigger prompt injected a backlog ID (e.g., `SL-003`), use the matching promoted form (`SL-003` → `SW-003`). Same number, `SL` (backlog) → `SW` (watchlist).
   - If no backlog ID was injected, generate a slug: `AUTO-{CHANNEL}-{first 3 words of SIGNAL_NAME, kebab-case}`, e.g., `AUTO-sector-rotation-xly-xlp-gate`.
   - For AUTO-DROP and NEEDS_REVIEW, emit the SIGNAL_ID anyway (tracking purposes), but morning will ignore it.
7. Record: `DECISION`, `CHANNEL`, `SIGNAL_ID`, `SIGNAL_NAME`, `PERFORMANCE`, `CREATIVITY`, `CHECK_RULE`, `FINDING_SUMMARY`.

**If research fails or times out:** skip this step entirely. Do NOT emit a SIGNAL_RESULT block. Note "Signal research skipped this session" in the email body.

## Step 7 — Email Assembly

**Subject template (normal):** `EOD Report — [Date] — [N] positions, [+/-]X.XX%, [stop-out summary], research [outcome]`

**Subject template (degraded):** `EOD Report — [Date] — DEGRADED ([failure_class]), [N] positions [†], research [outcome]`

The DEGRADED tag in the subject is load-bearing — tomorrow's morning agent (which pulls "EOD Report" emails for fallback data in its own Phase D) uses the DEGRADED tag to know that the snapshot is approximate, not authoritative.

Fill rules:
- `[Date]` → short day+date, e.g., `Wed Apr 22, 2026`
- `[N] positions` → current count (from `positions_snapshot_eod` if live; from morning email if degraded)
- `[+/-]X.XX%` → day P/L percent, signed (omit if degraded — no `last_equity`)
- `[stop-out summary]` → `"no stop-outs"` or `"N stop-out(s)"` (omit if degraded — no fills data)
- `[outcome]` → one of: `"promoted"` (AUTO-PROMOTE), `"dropped"` (AUTO-DROP), `"needs review"` (NEEDS_REVIEW), `"skipped"` (research_status=skipped)
- `[failure_class]` (degraded only) → `discovery_exhausted` / `render_down` / `alpaca_api_error`

Examples:
- `EOD Report — Wed Apr 22, 2026 — 8 positions, +0.30%, 1 stop-out, research needs review`
- `EOD Report — Fri Apr 24, 2026 — 7 positions, -0.85%, no stop-outs, research promoted`
- `EOD Report — Thu Apr 23, 2026 — 8 positions, +0.12%, no stop-outs, research skipped`

**Body structure (exact order, PLAINTEXT — no HTML):**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
END-OF-DAY REPORT — [Date]
Mode: READ-ONLY  (Mode: READ-ONLY-DEGRADED if Phase D fired)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[If degraded, insert this block immediately below the header:
🔌 CONNECTIVITY STATUS
Alpaca: OFFLINE | Failure class: [failure_class] | Discovery attempts: [N]/7 | Render status: [up/down/unprobed]
- If render_status = up: Runtime/connector issue, NOT Render. Reconnect at https://claude.ai/settings/connectors
- If render_status = down: Render endpoint unreachable. Check dashboard for alpaca-mcp-server-8zw5
- If render_status = unprobed: WebFetch unavailable; could not distinguish runtime vs Render
Position values below are approximate (qty × current_quote). No fills data, no stop-out detection. Cash frozen at this morning's open.
Tomorrow's morning agent will see DEGRADED in this email's subject — it will treat this snapshot as approximate, not authoritative.]

📊 ACCOUNT
Equity: $X  {sparkline}  Day: {sign}{abs%}%
Cash: $Y (Z%) | Invested: $A (B%)
Since inception (Apr 8): $E (F%) | Buying power: $G

[Sparkline rendering: pull last 7 trading days from get_portfolio_history. Map each day's equity to one of these blocks based on linear scale (min=▁, max=█): ▁▂▃▄▅▆▇█. Output as 7 chars, one per day, oldest to newest. Example: "▃▄▅▄▆▅▆" for a week ending slightly up. If portfolio_history failed, omit the sparkline entirely.]

📈 SESSION ACTIVITY
Fills today: [N total logical orders — list each with time, ticker, side, qty, price, source. Source uses the client_order_id prefix convention from Step 4.]
Realized P/L today: $X
Stop-outs: [list or "None"]

[Zero-activity day handling: if N = 0 fills, render this section as:
"No fills today. Portfolio unchanged from yesterday's close. Realized P/L: $0. Stop-outs: None."
The post-mortems section below is omitted entirely on zero-stop-out days.]

📋 DAY RECONCILIATION

Morning proposals ([N]):
| Ticker | Authority | Proposed | Status |
|--------|-----------|----------|--------|
| ...    | a/b/c/d   | qty@limit| FILLED / WORKING / NOT_EXECUTED |

Midday proposals ([N]):
| Ticker | Authority | Proposed | Status |
|--------|-----------|----------|--------|
| ...    |           |          | |

[Omit either section if that agent proposed nothing.]

🌍 TAPE (close of session)
SPY: $X (+Y% day) | QQQ: $X (+Y%) | IWM: $X (+Y%) | VIX: X.X
Day narrative (1-2 sentences on what drove the tape): [brief]

📁 PORTFOLIO SNAPSHOT

| Ticker | Qty | Avg Cost | Close | Day Chg | Unrealized P/L | % Equity | 🔔 |
|--------|-----|----------|-------|---------|----------------|----------|-----|

Position P/L (visual):
[For each position, render one line: `{emoji} {TICKER}  {bars}{padding} {sign}{abs%}%`
- emoji: 🟢 if unrealized_plpc > 0, 🔴 if < 0, ⚪ if exactly 0
- bars: full block "█" repeated round(abs(unrealized_plpc * 100) / 1) times, MAX 10 (one block per 1% gain/loss, capped at 10%)
- padding: spaces to right-pad bars to 10 chars
- Example: `🟢 NVDA  ██████████ +10.1%`
- Example: `🔴 XLF   █          -1.1%`
Sort by abs(unrealized_plpc) descending — biggest movers first.]

🛑 TRAIL STOP DASHBOARD

| Ticker | Qty | Trail % | Stop | HWM | Headroom |
|--------|-----|---------|------|-----|----------|

Stop headroom (visual):
[For each position with a trail stop, render: `{TICKER}  [{filled}{empty}] {headroom}% to stop {warn}`
- filled: full block "█" repeated round(headroom_pct / 0.5) times, MAX 10 (one block per 0.5% headroom, capped at 5%)
- empty: light shade "░" filling remainder to 10 chars total inside brackets
- warn: "⚠" if headroom < 2%, else ""
- Example: `SPY   [█████████░] 4.5% to stop`
- Example: `XLI   [███░░░░░░░] 1.8% to stop ⚠`
Sort by headroom ascending — riskiest stops first.]

📓 POST-MORTEMS
[Any stop-outs today — one block per ticker per Step 5 template. Omit section if no stop-outs.]

🔬 SIGNAL RESEARCH
[If research ran — findings summary (2-3 sentences), grades, recommendation. If skipped — "Signal research skipped this session" with reason (timeout / error / etc.).]

[If research produced a result, include the machine-readable block below. If research was skipped, omit the block entirely — morning handles missing blocks.]

<!-- SIGNAL_RESULT_START
DECISION: [AUTO-PROMOTE or AUTO-DROP or NEEDS_REVIEW]
CHANNEL: [channel name, kebab-case]
SIGNAL_ID: [SW-XXX if promoted from SL-XXX backlog, else AUTO-{channel}-{slug}]
SIGNAL_NAME: [short name]
PERFORMANCE: [0-5]
CREATIVITY: [0-5]
CHECK_RULE: [full mechanically-evaluable rule on a single line if promoting, else "N/A"]
FINDING_SUMMARY: [one sentence]
SIGNAL_RESULT_END -->

━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Next scheduled run: Morning Trading Plan, tomorrow 9:00 AM ET.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## Step 8 — Create Gmail Draft (dual-format)

### Pre-flight: SIGNAL_RESULT block validation

If Step 6 produced a SIGNAL_RESULT block, validate BEFORE calling `create_draft`:
- All 8 fields present and non-empty: `DECISION`, `CHANNEL`, `SIGNAL_ID`, `SIGNAL_NAME`, `PERFORMANCE`, `CREATIVITY`, `CHECK_RULE`, `FINDING_SUMMARY`
- `DECISION` is exactly one of `AUTO-PROMOTE`, `AUTO-DROP`, `NEEDS_REVIEW` (case-sensitive, hyphens/underscores as shown)
- `PERFORMANCE` and `CREATIVITY` are integers 0-5
- `SIGNAL_ID` is non-empty (format `SW-XXX` if promoted from backlog, else `AUTO-{channel}-{slug}`)
- `CHECK_RULE` is `N/A` when `DECISION != AUTO-PROMOTE`, else a complete mechanically-evaluable rule on a single line (NO embedded newlines — parsers rely on each field being on one line)
- Opening delimiter is exactly `<!-- SIGNAL_RESULT_START` on its own line
- Closing delimiter is exactly `SIGNAL_RESULT_END -->` on its own line

**If ANY check fails, omit the entire block from BOTH bodies.** A partial or malformed block will break morning's Step 1.5 parser and produce silent signal errors.

### Construct HTML body

Build an HTML version of the email body that mirrors the plaintext sections from Step 7 but renders as a styled document. Use the visual conventions in Appendix D. Include the `<!-- SIGNAL_RESULT_START ... -->` block in the HTML body verbatim too — HTML comments survive HTML rendering invisibly and provide a redundant copy in case future tooling parses the HTML body instead of plaintext.

The HTML body should include:
- Section headers with the emoji + styled `<h2>` (per Appendix D)
- Account summary as a styled box (Appendix D's "summary box" pattern)
- Portfolio snapshot as a real `<table>` with color-coded P/L cells
- Position P/L visual rendered with HTML/CSS bars (rather than Unicode blocks) — see Appendix D
- Trail stop dashboard as a styled `<table>` with color-coded headroom cells (red <2%, amber 2-3%, green >3%)
- Signal research as a styled "card" div
- Same emoji section anchors as plaintext (📊 📈 📋 🌍 📁 🛑 📓 🔬) so it's visually familiar

### Draft creation

Call Gmail `create_draft` with:
- **To:** `{{RECIPIENT_EMAIL}}`
- **Subject:** per Step 7
- **body:** the plaintext body per Step 7. **MUST remain plain text** — this is the load-bearing contract with morning's parser via `plaintextBody`.
- **htmlBody:** the styled HTML body constructed above. This is what Dylan sees rendered in Gmail.

Both fields are required.

### Failure handling for create_draft

If `create_draft` returns an error specifically related to the htmlBody (malformed HTML, validation rejection, etc.):
1. Retry once WITHOUT the htmlBody parameter, sending only the plaintext `body`.
2. Note in the trigger run log: "HTML body rejected; sent plaintext only."
3. Do not let HTML failure block the report from shipping.

If Gmail MCP fails entirely: log the full plaintext body to the trigger's run output so Dylan can find it in the Anthropic trigger logs.

### Post-create verification (catch silent connector failures)

The Gmail connector has been observed returning a fake-success response from `create_draft` without persisting the draft (incident 2026-05-01 — all three agents reported "Gmail draft created successfully" but no drafts landed in Gmail; suspected root cause was OAuth invalidation following a Google password rotation, leaving the connector returning success silently). Verify before exiting:

1. Capture the `draftId` returned by `create_draft` and the exact `Subject` string you used.
2. Sleep 5 seconds — Gmail's draft indexing has minor lag.
3. Call `list_drafts` with `query: "subject:\"<exact subject>\""`, `pageSize: 5`. Treat as a match only if a returned draft's `date` is within the last 5 minutes.
4. **≥1 matching recent draft:** verification passes. Proceed to Step 9.
5. **Zero matches:** the connector silently failed. Retry `create_draft` once with identical parameters. Sleep 5s. Re-run the `list_drafts` probe.
6. **Still zero matches after retry:** output to trigger run log:
   ```
   DRAFT_VERIFY_FAILED | subject: <subject> | claimed_id_1: <id1> | claimed_id_2: <id2 or "n/a"> | Gmail connector likely broken — reconnect at https://claude.ai/settings/connectors before next run
   ```
   Then EXIT WITH AN ERROR (non-zero). The trigger schedule UI must show RED for this run — green-with-no-draft is the exact failure mode this step exists to surface. (EOD's failure-modes table in Appendix A makes one exception: Alpaca-unreachable is non-fatal. Gmail-fake-success is fatal — this overrides the Appendix A "EOD should always ship something" guidance, because in this case it didn't ship anything despite claiming to.)

## Step 9 — Completion

Output a single status line:
```
EOD report drafted | Stop-outs: [N] | Research: [DECISION or "skipped"] | Equity: $X | Status: OK
```

Exit.

---

## Appendix A — Failure Modes (non-fatal, report still ships)

The EOD agent ships the report even when individual steps fail. Log the failure in the email body's relevant section and continue. Updated 2026-05-10 to distinguish fail classes that go to Phase D vs. ones that just degrade specific sections.

| Failure | Triggers Phase D? | Report impact | Continue? |
|---------|-------------------|---------------|-----------|
| Step 1 Phase A discovery exhausted (7 attempts) | Yes | DEGRADED report from morning email + current quotes; no reconciliation; no post-mortems; signal research still runs | Yes |
| Step 1 Phase A Render probe → "down" | Yes | Same as above | Yes |
| Step 1 Phase B `get_clock` fails twice | Yes | Same as above | Yes |
| Step 1 Phase D itself fails (no morning email + no quotes) | No (true standdown) | True standdown — minimal email noting both failures | Exit |
| Morning email not found (live mode only — Phase D requires it) | No | Reconciliation section shows "no morning email parsed" | Yes |
| Midday email not found | No | Same, for midday section | Yes |
| `get_portfolio_history` fails | No | Skip since-inception curve, use `last_equity` for day P/L | Yes |
| Signal research timeout | No | Omit SIGNAL_RESULT block, note "skipped" | Yes |
| Signal research errors mid-search | No | Same as timeout | Yes |
| Gmail create_draft fails AND verification confirms no draft landed | No | Log full body to trigger output, EXIT non-zero so trigger UI shows red | **Fatal for reporting; the silent-fake-success failure mode IS the bug we're catching** |

**Fatal errors (abandon and exit):** Step 1 Phase D failing entirely (no morning email + no live quotes), and Gmail draft-create + post-verify both failing. Everything else degrades gracefully.

The "EOD should always ship something" principle holds with one caveat: a confirmed-failed Gmail post-verify is fatal because shipping nothing-but-claiming-success is worse than visibly red in the trigger UI. See Step 8 post-create verification.

## Appendix B — Signal Target Rotation

The `Signal target` is injected via the trigger prompt. Edit the trigger prompt (not this playbook) to rotate targets. This keeps the playbook stable and lets Dylan rotate hypotheses weekly without git commits.

**Current backlog** (from `notes/Process Audit & Signal Lab Design - April 16.md`):
- SL-001: insider-activity — cluster insider buying in semis predicts NVDA moves
- SL-002: options-flow — unusual call sweeps on mega-cap tech as leading indicator
- SL-003: sector-rotation — XLY/XLP ratio as risk-on/off regime signal
- SL-004: commodity-signals — copper futures curve for FCX
- SL-005: macro-surprise — data surprise index predicts sector rotation
- SL-006: sentiment-positioning — AAII + fund flows as contrarian indicator
- SL-007: earnings-revision — upward revision momentum as entry signal
- SL-008: volatility-regime — VIX term structure as market regime indicator

When the current target reaches AUTO-PROMOTE or AUTO-DROP, rotate to the next priority-1 item.

## Appendix C — SIGNAL_RESULT Block Contract

This block is parsed by morning's Step 1.5. **Do not change labels, delimiters, or line format without coordinating a morning playbook update.**

Fields (all required when a block is emitted):
- `DECISION`: one of `AUTO-PROMOTE`, `AUTO-DROP`, `NEEDS_REVIEW`
- `CHANNEL`: short kebab-case identifier (e.g., `insider-activity`)
- `SIGNAL_NAME`: human-readable name (e.g., `Insider Buying Clusters`)
- `PERFORMANCE`: integer 0-5
- `CREATIVITY`: integer 0-5
- `CHECK_RULE`: one sentence, mechanically evaluable. `N/A` if DECISION is not AUTO-PROMOTE.
- `FINDING_SUMMARY`: one sentence summary of the investigation

The block's delimiters are HTML comments so they render invisibly in HTML clients but survive plaintext extraction. Preserve `<!-- SIGNAL_RESULT_START` and `SIGNAL_RESULT_END -->` exactly.

## Appendix D — HTML Visual Conventions (for htmlBody)

Use these consistently across all sections. Inline styles (no `<style>` blocks — Gmail strips those).

### Color tokens
- Positive values (gains, %s up): `color: #16a34a` (green)
- Negative values (losses, %s down): `color: #dc2626` (red)
- Neutral / info gray: `color: #6b7280`
- Warning amber: `color: #eab308`
- Section header dark: `color: #1e293b`
- Accent blue (header underline): `#3b82f6`

### Section header
```html
<h2 style="font-family:Arial,sans-serif;color:#1e293b;border-bottom:2px solid #3b82f6;padding-bottom:4px;margin-top:24px;">📊 Section Title</h2>
```

### Account summary box
```html
<div style="background:#f8fafc;border:1px solid #e2e8f0;border-radius:8px;padding:16px;margin:12px 0;font-family:Arial,sans-serif;">
  <div style="font-size:24px;font-weight:bold;">$101,584.58 <span style="font-size:14px;font-weight:normal;color:#6b7280;">(7-day curve: ▃█▇▃▅▁▃)</span></div>
  <div style="color:#16a34a;font-size:16px;">Day: +$640 (+0.60%)</div>
  <div style="color:#16a34a;font-size:14px;">Since Inception (Apr 8): +$1,584.58 (+1.58%)</div>
  <div style="color:#6b7280;margin-top:4px;">Cash: $46,501 (45.8%) | Invested: $55,083 (54.2%) | Buying Power: $148,086</div>
</div>
```

### Standard table
```html
<table style="width:100%;border-collapse:collapse;font-family:Arial,sans-serif;font-size:14px;margin:8px 0;">
  <thead>
    <tr style="background:#1e293b;color:white;">
      <th style="padding:8px;text-align:left;">Ticker</th>
      <th style="padding:8px;text-align:right;">Qty</th>
      <th style="padding:8px;text-align:right;">Close</th>
      <th style="padding:8px;text-align:right;">P/L</th>
    </tr>
  </thead>
  <tbody>
    <!-- Alternate row backgrounds: #ffffff and #f8fafc -->
    <tr style="background:#ffffff;">
      <td style="padding:8px;">SPY</td>
      <td style="padding:8px;text-align:right;">22</td>
      <td style="padding:8px;text-align:right;">$714.05</td>
      <td style="padding:8px;text-align:right;color:#16a34a;">+$612.14</td>
    </tr>
  </tbody>
</table>
```

### P/L bar (HTML version of plaintext Unicode bar)
Render each position's P/L as a horizontal styled `<div>` with width proportional to `abs(unrealized_plpc * 10)` percent, max 100%. Positive = green fill, negative = red fill.

```html
<div style="display:flex;align-items:center;gap:8px;font-family:Arial,sans-serif;font-size:14px;margin:2px 0;">
  <span style="display:inline-block;width:50px;font-weight:bold;">NVDA</span>
  <div style="flex:1;background:#f1f5f9;height:14px;border-radius:2px;overflow:hidden;max-width:300px;">
    <div style="width:100%;height:100%;background:#16a34a;"></div>
  </div>
  <span style="color:#16a34a;font-weight:bold;width:60px;text-align:right;">+14.58%</span>
</div>
```

### Stop headroom gauge
Same pattern as P/L bar but use color thresholds: red bar if <2%, amber if 2-3%, green if >3%.

### Signal research card
```html
<div style="background:#eff6ff;border:1px solid #bfdbfe;border-radius:8px;padding:16px;margin:12px 0;font-family:Arial,sans-serif;">
  <div style="font-size:16px;font-weight:bold;color:#1e40af;">🔬 Signal Research: [SIGNAL_NAME]</div>
  <div style="color:#374151;margin-top:8px;font-style:italic;">[Hypothesis]</div>
  <div style="margin-top:12px;">
    <span style="font-weight:bold;">Performance:</span> X/5 &nbsp;|&nbsp;
    <span style="font-weight:bold;">Creativity:</span> X/5 &nbsp;|&nbsp;
    <span style="font-weight:bold;color:[#16a34a for PROMOTE / #6b7280 for REVIEW / #dc2626 for DROP];">[DECISION]</span>
  </div>
  <div style="color:#374151;margin-top:8px;">[FINDING_SUMMARY]</div>
</div>
```

### Body wrapper
Wrap the entire HTML body in a max-width container so it renders cleanly in wide email windows:
```html
<body style="font-family:Arial,sans-serif;max-width:720px;margin:0 auto;padding:16px;">
  <!-- all sections here -->
</body>
```

### Rendering rules
- **No external images** unless explicitly added (chart embedding deferred — see project memory)
- **No `<style>` blocks** — Gmail strips them; use inline styles only
- **No JavaScript** — Gmail strips it
- **Mirror the plaintext section order exactly** — easier for Dylan to compare formats if he wants
- **Preserve emoji in HTML** — Gmail renders them as native emoji

---

*End of playbook. The agent should exit after Step 9.*
