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

EOD does NOT check for the `TRADING-PAUSED` kill switch. The kill switch pauses trading; EOD only reports. Reports are always useful — run every day regardless.

---

## Step 1 — Health Check

1. Call Alpaca `get_clock`. If it fails, wait ~30 seconds and retry once.
2. If still failing: still produce a report. Set `alpaca_available = false` and continue. The email will note "Alpaca unreachable — report omits live data."
3. If succeeds: call `get_account_info`. Record baseline: `equity`, `cash`, `last_equity`, `portfolio_value`, `buying_power`.
4. Record market state — was the market open today? If holiday / closed, note it but still produce a minimal report (no trading activity means a short email, that's fine).

## Step 2 — Parse Today's Morning & Midday Emails

Pull the day's narrative from the agents that already ran:

1. **Morning:** search Gmail with
   ```
   from:{{RECIPIENT_EMAIL}} (subject:"Morning Trading Plan" OR subject:"Morning Brief") newer_than:1d
   ```
   Take the most recent. Call `get_thread` with `messageFormat: FULL_CONTENT`. Extract from `plaintextBody`:
   - Proposed tickers + authority
   - Considered-but-rejected tickers
   - Flagged positions
   - Overnight events
   - **If `plaintextBody` is empty or contains only a shell string like "see HTML version":** set `morning_parse_status = unparseable`. Do NOT fail. Fall back to the `client_order_id` prefix convention (Step 4) for reconciliation.
2. **Midday:** search Gmail with
   ```
   from:{{RECIPIENT_EMAIL}} subject:"Midday Brief" newer_than:1d
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

**Subject template:** `EOD Report — [Date] — [N] positions, [+/-]X.XX%, [stop-out summary], research [outcome]`

Fill rules:
- `[Date]` → short day+date, e.g., `Wed Apr 22, 2026`
- `[N] positions` → current count from `positions_snapshot_eod`
- `[+/-]X.XX%` → day P/L percent, signed
- `[stop-out summary]` → `"no stop-outs"` or `"N stop-out(s)"`
- `[outcome]` → one of: `"promoted"` (AUTO-PROMOTE), `"dropped"` (AUTO-DROP), `"needs review"` (NEEDS_REVIEW), `"skipped"` (research_status=skipped)

Examples:
- `EOD Report — Wed Apr 22, 2026 — 8 positions, +0.30%, 1 stop-out, research needs review`
- `EOD Report — Fri Apr 24, 2026 — 7 positions, -0.85%, no stop-outs, research promoted`
- `EOD Report — Thu Apr 23, 2026 — 8 positions, +0.12%, no stop-outs, research skipped`

**Body structure (exact order, PLAINTEXT — no HTML):**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
END-OF-DAY REPORT — [Date]
Mode: READ-ONLY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📊 ACCOUNT
Equity: $X | Cash: $Y (Z%) | Invested: $A (B%)
Day P/L: $C (D%) | Since inception (Apr 8): $E (F%)
Buying power: $G

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

🛑 TRAIL STOP DASHBOARD

| Ticker | Qty | Trail % | Stop | HWM | Headroom |
|--------|-----|---------|------|-----|----------|

[Any stop with headroom <2% gets a ⚠ mark in Headroom column.]

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

## Step 8 — Create Gmail Draft

### Pre-flight: SIGNAL_RESULT block validation

If Step 6 produced a SIGNAL_RESULT block, validate BEFORE calling `create_draft`:
- All 8 fields present and non-empty: `DECISION`, `CHANNEL`, `SIGNAL_ID`, `SIGNAL_NAME`, `PERFORMANCE`, `CREATIVITY`, `CHECK_RULE`, `FINDING_SUMMARY`
- `DECISION` is exactly one of `AUTO-PROMOTE`, `AUTO-DROP`, `NEEDS_REVIEW` (case-sensitive, hyphens/underscores as shown)
- `PERFORMANCE` and `CREATIVITY` are integers 0-5
- `SIGNAL_ID` is non-empty (format `SW-XXX` if promoted from backlog, else `AUTO-{channel}-{slug}`)
- `CHECK_RULE` is `N/A` when `DECISION != AUTO-PROMOTE`, else a complete mechanically-evaluable rule on a single line (NO embedded newlines — parsers rely on each field being on one line)
- Opening delimiter is exactly `<!-- SIGNAL_RESULT_START` on its own line
- Closing delimiter is exactly `SIGNAL_RESULT_END -->` on its own line

**If ANY check fails, omit the entire block.** A partial or malformed block will break morning's Step 1.5 parser and produce silent signal errors. Better to emit no block and let morning treat it as "no fresh signal available."

### Draft creation

Call Gmail `create_draft` with:
- **To:** `{{RECIPIENT_EMAIL}}`
- **Subject:** per Step 7
- **Body:** per Step 7 — **MUST be plain text**, not HTML. The next morning agent's Step 1.5 parses this body via `plaintextBody`, especially the `<!-- SIGNAL_RESULT_START ... -->` block. HTML-only bodies will be unparseable.

The SIGNAL_RESULT block must be preserved EXACTLY — same field labels, same delimiters. This is the load-bearing contract with morning.

If Gmail MCP fails: log the full email body to the trigger's run output so Dylan can find it in the Anthropic trigger logs.

## Step 9 — Completion

Output a single status line:
```
EOD report drafted | Stop-outs: [N] | Research: [DECISION or "skipped"] | Equity: $X | Status: OK
```

Exit.

---

## Appendix A — Failure Modes (non-fatal, report still ships)

The EOD agent ships the report even when individual steps fail. Log the failure in the email body's relevant section and continue.

| Failure | Report impact | Continue? |
|---------|---------------|-----------|
| Alpaca unreachable after retry | Report omits live data, reconciliation skipped | Yes — ship minimal report noting outage |
| Morning email not found | Reconciliation section shows "no morning email parsed" | Yes |
| Midday email not found | Same, for midday section | Yes |
| `get_portfolio_history` fails | Skip since-inception curve, use `last_equity` for day P/L | Yes |
| Signal research timeout | Omit SIGNAL_RESULT block, note "skipped" | Yes |
| Signal research errors mid-search | Same as timeout | Yes |
| Gmail create_draft fails | Log full body to trigger output | **Fatal for reporting; continue to research if not yet done** |

Fatal errors (abandon and exit): none. EOD should always ship something.

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

---

*End of playbook. The agent should exit after Step 9.*
