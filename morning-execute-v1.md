# Morning Trading Playbook — Execute v1

*Paste this into `github.com/Dylanxmt/finance-playbooks/morning-execute-v1.md`. Version: 2026-05-10 (initial — graduated from `morning-dryrun-v1.md` after dry-run validation).*

*This file is fetched at runtime by the Anthropic scheduled trigger. **This playbook executes real orders against the configured Alpaca account.** Verify `ACCOUNT_ID` resolves to a paper account before adopting on the trigger.*

---

## 🟢 EXECUTE MODE — what changed from dry-run

This is the live-action successor to `morning-dryrun-v1.md`. The decision logic (Steps 0-7) is identical and cross-referenced. The change is at Step 8: candidates that pass guardrails become actual `place_stock_order` calls plus immediate trail-stop follow-ups.

**Allowed order tools** (only these — never any others):
- `place_stock_order` — for buys (authorities a, b) and sells (authority c)
- `cancel_order_by_id` — only to cancel a stale morning-placed limit that timed out, or to replace a trail after a trim fill
- `replace_order_by_id` — only for adjusting trail stops mid-flow (e.g., re-sizing after a trim)

**Forbidden in morning execute** (explicitly):
- `place_crypto_order`, `place_option_order` — separate playbooks if/when needed
- `close_position`, `close_all_positions` — morning never exits a full position; trail stops own that
- `exercise_options_position`, `do_not_exercise_options_position` — N/A
- `update_account_config` — never modify account settings from an agent

If you catch yourself about to call a forbidden tool, STOP. Finalize the email with what was placed (or attempted) and exit.

---

## Context injected by trigger

- **Mode:** EXECUTE
- **Approval mode:** `{{APPROVAL_MODE}}` — one of `auto` (default) or `manual`. In `manual`, the playbook builds the order plan + Gmail draft but does NOT call any place/replace/cancel tool. Use `manual` for the first 1-2 weeks of execute mode if you want a final human gate, then flip to `auto`. **For live trading (post hardening checklist item 2), `manual` is mandatory for the first 4 weeks.**
- **Target allocation:** 70% invested / 30% cash (hard floor: 25% cash — never breach)
- **Account:** Alpaca `{{ACCOUNT_ID}}` (must be paper for now — see hardening checklist before live)
- **Recipient:** `{{RECIPIENT_EMAIL}}`
- **Today:** the date the trigger fires

## Risk-tier reference (same as dry-run)

| Tier | Trail % | Examples |
|------|---------|----------|
| Stable/defensive | 3% | XLP, XLU, JPM, XLF, XLV, KO, PG, VZ |
| Moderate | 4% | SPY, IWM, GLD, HD, XLE, XLI, XLB, XLK, sector ETFs generally |
| Volatile/growth | 5% | NVDA, META, FCX, AMD, TSLA, small caps, momentum single-names |

## Hard caps (enforced in Step 7)

- Max per single trade: **5% of equity** (live tightens to 2-3%)
- Max NEW positions opened per day (authority a): **3**
- Max total order submissions per session: **5** (3 new + up to 2 top-ups/trims/tighten combined)
- Min cash reserve: **25% of equity** post-execution (never breach)
- Order types: **LIMIT only** — `day` time-in-force, never market, never extended hours
- Every new position MUST have a trail stop placed within 5 minutes of fill

---

## Step 0 — Kill Switch Check

Identical to `morning-dryrun-v1.md` Step 0 (two paths: repo file + Gmail label, fail-closed if both unreachable). Do not duplicate logic — apply the same sequence.

**Execute-mode hardening:** past Step 0, the agent has authority to place real orders. Step 0c (both paths unreachable) MUST fail closed in execute mode — refuse to proceed without a confirmed safety signal.

## Step 1 — Health Check (discovery-tolerant + degraded-mode fallback)

Identical to `morning-dryrun-v1.md` Step 1 (Phases A/B/C/D). Key execute-mode facts:
- If Phase D fires (degraded mode), `alpaca_available = false` — **NO orders may be placed** in degraded mode (order plumbing unreachable). Email reports degraded state with the Connectivity Status block and exits without touching any order tool.
- If Phase C account check fails (account blocked) → true standdown, no orders.
- Only proceed past Step 1 with `alpaca_available = true` and `position_source = "alpaca-live"`.

## Step 1.5 — Active Signal Watchlist (from EOD research)

Identical to `morning-dryrun-v1.md` Step 1.5. Parse the most recent EOD email's `SIGNAL_RESULT` block. **One execute-mode addition:** if the most recent EOD email's subject contains `DEGRADED`, the EOD agent ran in degraded mode — its portfolio snapshot is approximate. Continue to parse the signal block (signals are independent of Alpaca) but do NOT use the degraded EOD's portfolio table for any reconciliation.

## Step 2 — Portfolio Snapshot

Identical to `morning-dryrun-v1.md` Step 2 with one execute-mode addition:

**Reconcile against yesterday's placed orders:**
- For each `client_order_id` in yesterday's EOD email's "💼 ORDERS PLACED" section (when present — only EOD-execute writes this), check via `get_order_by_client_id` whether the order ultimately filled, was cancelled, or expired.
- For overnight trail stops with `time_in_force = "gtc"`: they should still be working unless they fired. Reconcile against today's `get_orders` with `status: open` and today's `get_account_activities` for any overnight fills.
- Record `morning_overnight_reconciliation` for the email's "🔔 OVERNIGHT EVENTS" section.

## Step 3 — Market Context

Identical to `morning-dryrun-v1.md` Step 3.

## Step 4 — Position-Level Scan

Identical to `morning-dryrun-v1.md` Step 4.

## Step 5 — Sector Rotation Analysis

Identical to `morning-dryrun-v1.md` Step 5.

## Step 6 — Opportunity Scan

Identical to `morning-dryrun-v1.md` Step 6. Generates up to 5 candidates across authorities (a) new position / (b) top-up / (c) trim / (d) rebalance.

## Step 7 — Guardrail Filter (HARD)

Identical to `morning-dryrun-v1.md` Step 7 (two-pass: per-candidate + cumulative). **Execute-mode hardening:** each check uses LIVE values, not estimates. Pre-execution sanity in Step 8a re-validates using fresh quotes immediately before placement.

## Step 8a — Pre-Execution Sanity

Run these checks ONCE just before the first order call. If any fails, abort the entire order batch (skip 8b/8c/8d) and produce a "near-miss" email noting what failed.

For each surviving candidate:
1. Fetch `get_stock_latest_quote` for the ticker. Confirm the quote is fresh (`timestamp` within last 60s if market is open; pre-market is OK if `is_open = false`).
2. Confirm bid/ask spread is sane: `(ask − bid) / mid < 0.5%` for liquid names, `< 1.0%` for small-caps. Wider than that → defer this candidate (note as "spread too wide for safe limit"). Note: pre-market spreads are typically wider — use the previous close's spread as a sanity reference if pre-market spread looks abnormal.
3. Call `get_clock`. If `is_open = false`, that's expected at 9:00-9:30 ET — proceed with limit orders that will sit until open. If `is_open = false` AND it's past 10:00 ET (e.g., the agent ran late), something is wrong — abort all orders, log the clock state, exit.
4. Re-check the cumulative cash floor: `cash − sum(all candidate costs) >= equity * 0.26` (1pp safety margin above the 25% floor). If breached, drop the lowest-conviction candidate and re-check. Repeat until set passes or empty.
5. For top-ups (authority b): confirm the underlying position still exists in `get_all_positions` and is still <10% of equity post-topup.

If `APPROVAL_MODE = manual`: skip Step 8b/8c — write the order plan into the email under "💼 PROPOSED FOR YOUR APPROVAL" using the rationale template from `morning-dryrun-v1.md` Step 9, then exit at Step 10 (email assembly) with subject `Morning Trading Plan — [Date] — [N] proposed for approval`.

## Step 8b — Order Placement

Place orders one at a time in this priority order (NEVER parallelize order calls — race conditions on cash math):

1. **Trims (authority c)** — sells, free cash for downstream actions. Use `place_stock_order` (sell, limit, day).
2. **Top-ups (authority b)** — buys, use cash. Use `place_stock_order` (buy, limit, day).
3. **New positions (authority a)** — buys, use remaining cash. Use `place_stock_order` (buy, limit, day).

**Authority (d) rebalance** — manifests as either an additional (a)/(b) or a (c) depending on direction. Treat as the appropriate category above.

For each order:
- Compute limit price per Appendix E.
- Generate `client_order_id` per Appendix B2.
- Call `place_stock_order`.
- Capture the returned `id`, `client_order_id`, submitted timestamp.
- If the call errors: log the error with the candidate's full context. Skip this candidate, do NOT retry. Continue to the next candidate.
- If a trim fills almost immediately (pre-market sometimes does on a wide-spread crossing), the cash from that fill becomes available for the next downstream candidate — re-check the cumulative cash floor before each subsequent submission.

Hold all submitted orders as `submitted_orders[]` for Step 8c and 8d.

## Step 8c — Trail Stop Queuing

Each authority needs a specific trail-stop strategy. Queue trails here; they get placed in Step 8d after the parent order fills (or right away for trims that need stop adjustment).

**Authority (a) — new position:**
- After the buy fills, place a brand-new trail stop covering the full filled qty.
- Trail % per the risk-tier table.
- `client_order_id`: `morning-{YYYY-MM-DD}-{ticker}-stop-{seq}`.

**Authority (b) — top-up:**
- After the buy fills, place a SEPARATE trail stop for ONLY the newly purchased shares (`filled_qty` from the top-up, not the full combined position).
- The original position's existing trail is left untouched — its HWM (potentially well above current price) continues to protect the original shares.
- This intentionally creates two trails per ticker until the original eventually exits. EOD reconciliation handles dual-trail attribution by `client_order_id` prefix.

**Authority (c) — trim:**
- After the sell fills, CANCEL the existing trail stop for this ticker (`cancel_order_by_id`) and place a new trail for the reduced qty.
- Trail % unchanged from the existing trail. HWM resets to current price at placement (this is unavoidable — Alpaca trail stops re-anchor on placement).
- `client_order_id` for the replacement trail: `morning-{YYYY-MM-DD}-{ticker}-stop-{seq}` with a fresh sequence number.

## Step 8d — Fill Verification + Trail Placement

For each entry in `submitted_orders[]`, poll `get_order_by_id` until terminal:

- Poll cadence: every 30 seconds.
- Wait window: from submission until 5 minutes after market open (so a 9:02 submission has a deadline of 9:35).
- Terminal states: `filled`, `partially_filled` (after expiration), `canceled`, `expired`, `rejected`, `replaced`.

On terminal:
- `filled`: record `filled_avg_price`, `filled_qty`, `filled_at`. Place the queued trail stop from Step 8c using `filled_qty` (not originally submitted qty).
- `partially_filled` after expiration: cancel any remainder via `cancel_order_by_id`, record what filled. Place trail for the actually-filled qty (zero qty → skip trail; log "partial fill, no trail placed").
- `canceled` / `expired` / `rejected` / `replaced`: log the reason. Do NOT retry.
- Still working at deadline: cancel via `cancel_order_by_id`. Log as "timed out — limit too far from NBBO or thin liquidity. No trail placed."

After all parent orders are terminal, verify `get_account_info`: confirm `cash >= equity * 0.25` (the hard floor). If somehow breached (shouldn't happen if Step 8a math was right), log `CASH_FLOOR_BREACH` to the email loud red — this is a contract violation worth investigating.

## Step 9 — Rationale Template

Same as `morning-dryrun-v1.md` Step 9 (90-110 word ceiling per trade, `🎯/💡/✅/🔬/📏/🛑/🚫` block). **One execute-mode change:** the rationale now describes EXECUTED actions, not proposals. Replace "═══ TRADE #X OF Y" header with "═══ ORDER #X OF Y — [STATUS]" where STATUS is `FILLED $X.XX` / `PARTIAL N/M` / `WORKING` / `CANCELED` / `REJECTED` / `TIMED OUT`.

## Step 10 — Email Assembly

**Subject format:**
- Orders executed (`APPROVAL_MODE = auto`): `Morning Trading Plan — [Date] — [N] orders placed, [M] filled`
- Manual approval: `Morning Trading Plan — [Date] — [N] proposed for approval`
- Standdown — zero candidates passed guardrails: `Morning Brief — [Date] — Standing down (zero candidates passed guardrails)`
- Standdown — degraded mode: `Morning Brief — [Date] — DEGRADED ([failure_class])`
- Standdown — other reasons: `Morning Brief — [Date] — Standing down ([brief reason])`

**Body structure (exact order, omit empty sections):**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
MORNING TRADING PLAN — [Date]
Mode: EXECUTE | Approval: [auto/manual]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📊 ACCOUNT
[same as dry-run Step 10 — equity, cash, invested, day change, since inception, buying power]

🔔 OVERNIGHT EVENTS
- [stop-outs detected overnight]
- [reconciliation of yesterday's still-working orders that filled/cancelled/expired overnight]

🧭 ACTIVE SIGNALS (from EOD research)
[same as dry-run]

🌍 MARKET CONTEXT
[same as dry-run — signals from Step 3, sector rotation from Step 5, pair-delta interpretation]

📁 PORTFOLIO SNAPSHOT
[same table + position P/L visual + stop headroom visual as dry-run]

💼 ORDERS PLACED ([N])
[For each submitted_order, render a rationale card per Step 9 with the STATUS badge. Include:
- Ticker, side, authority, qty, limit price, client_order_id
- Fill status: filled @ $X.XX (slippage vs limit: $0.XX) / partial N/M / working / cancelled / rejected / timed out
- Linked trail-stop client_order_id (for any filled buy)
- Rationale (90-110 words: signal, thesis, why, signal confirmation if applicable, size, stop, disqualifiers)]

🛑 TRAIL STOPS PLACED ([M])
[For each trail placed in Step 8d:
- Ticker, qty, trail %, current stop estimate, client_order_id, parent order client_order_id]

🗑️ CANDIDATES NOT EXECUTED
[Any candidate dropped in Step 7 (guardrails), 8a (sanity), 8b (submission error), or 8d (fill failure). Format: ticker — reason — specific failure point.]

📈 PORTFOLIO IMPACT (post-execution)
Cash: $X (was $Y at session start)
Invested: $A (was $B)
Position count: N → M
Sector mix changes: [+XLU, -FCX partial, etc.]

🛑 OPEN TRAIL STOPS (full inventory after execution)

| Ticker | Qty | Trail % | Stop | HWM | Headroom |
|--------|-----|---------|------|-----|----------|

━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ EXECUTE MODE — orders above are LIVE on Alpaca {{ACCOUNT_ID}}
This is paper trading. Midday will check fill status and reconcile against this plan.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

For standdown emails, use the dry-run standdown body structure with the appropriate subject.

For `APPROVAL_MODE = manual`, replace "ORDERS PLACED" with "PROPOSED FOR YOUR APPROVAL" — the rationale cards stay the same, but with no fill status (since no order was placed). Add a footer: "Reply YES to this thread, or open an interactive Claude session, to execute the plan."

## Step 11 — Create Gmail Draft (dual-format)

Same procedure as `morning-dryrun-v1.md` Step 11 — plaintext `body` + styled `htmlBody`, post-create verification via `list_drafts`, retry on silent connector failure, exit non-zero on verification failure.

HTML rendering: use the same yellow-amber rationale-card style with a STATUS badge in the top-right corner of each card:
- `filled`: green badge with fill price
- `partial`: amber badge with filled / submitted qty
- `cancelled` / `timed out` / `expired`: gray badge
- `rejected`: red badge
- `working`: blue badge (if poll loop ended before terminal — should be rare)

## Step 12 — Completion

Output a single status line:
```
Morning execute drafted | Orders placed: [N] | Filled: [M] | Trails placed: [P] | Equity: $X | Mode: EXECUTE/[approval] | Status: OK
```

Exit.

---

## Appendix A — Decision tree (5 terminal states)

**Orders Executed**: Step 1 succeeded, ≥1 candidate placed. Subject: "Morning Trading Plan — [N] orders placed, [M] filled."

**Proposed for Approval** (`APPROVAL_MODE = manual`): Step 1 succeeded, candidates ready, Dylan approves manually. Subject: "Morning Trading Plan — [N] proposed for approval."

**Monitoring / no candidates**: Step 1 succeeded but zero candidates passed Step 7 guardrails. Subject: "Standing down (zero candidates passed guardrails)."

**Degraded mode**: Step 1 Phase D fired. NO orders placed. Subject includes "DEGRADED."

**True standdown**: kill switch / account blocked / market closed / Phase D also failed / unrecoverable error. NO orders placed. Subject includes "STANDING DOWN."

## Appendix B — Safety Rails Specific to Morning Execute

- **No pre-market trading.** Limits placed at 9:00-9:30 sit until open. Use `time_in_force: day` always.
- **No options, crypto, or shorts.** Separate playbooks required.
- **No leverage beyond existing 2x margin.** Discard any candidate that would require more.
- **No position in a ticker that stopped out in the last 5 trading days.** Avoid chasing a just-broken thesis.
- **Flag earnings within 5 trading days** on any candidate. Prefer to avoid the name or note the earnings risk explicitly.
- **Hard stop at 5 total order submissions per session** (3 new + up to 2 top-ups/trims combined).
- **No order placement when `cash < 26% equity` pre-check** (1pp safety margin above 25% floor for slippage).
- **Cancel any morning limit that hasn't filled within 5 min of market open.** A stale morning limit on the book would distort the midday plan.
- **Trail stop within 5 min of fill** for new positions — non-negotiable. If trail placement fails, log a `STOP_MISSING` alert in the email and the trigger run log.

## Appendix B2 — `client_order_id` contract

Every `place_stock_order` / `replace_order_by_id` / `cancel_order_by_id` call MUST set `client_order_id` using:

```
morning-{YYYY-MM-DD}-{ticker}-{authority}-{seq}
```

- `{authority}` ∈ {`open`, `topup`, `trim`, `stop`}
  - `open`: new-position buy for authority (a)
  - `topup`: top-up buy for authority (b)
  - `trim`: trim sell for authority (c)
  - `stop`: trail stop (placed after any buy fill, or as replacement after a trim)
- `{seq}` = 3-digit zero-padded, per (date + authority + ticker), starts at `001`

Examples:
- `morning-2026-05-15-XLU-open-001` — new XLU position buy
- `morning-2026-05-15-XLU-stop-001` — trail stop placed after XLU fills
- `morning-2026-05-15-NVDA-topup-001` — top-up buy on existing NVDA
- `morning-2026-05-15-NVDA-stop-001` — separate trail for the topped-up shares (original NVDA trail untouched)
- `morning-2026-05-15-SPY-trim-001` — trim sell
- `morning-2026-05-15-SPY-stop-001` — replacement trail after the trim (existing SPY trail cancelled)

EOD reconciliation uses the prefix + authority to attribute each fill. NEVER re-use a `client_order_id` — Alpaca will reject the second call.

**Cross-agent attribution:** if midday later modifies a morning-placed trail (authority `tighten`), midday's replacement order uses the midday prefix (`midday-...-tighten-...`), not the morning prefix. EOD sees both and attributes the original placement to morning and the modification to midday.

## Appendix C — Jargon Glossary

Same as `morning-dryrun-v1.md` Appendix C.

## Appendix D — HTML Visual Conventions

Same as `morning-dryrun-v1.md` Appendix D + the order-status badge convention from Step 11.

## Appendix E — Limit Price Calculation

For each order, fetch `get_stock_latest_quote` immediately before submission (Step 8a got a fresh quote — reuse if <30s old, else re-fetch).

**Buy limits (authority a — new position, authority b — top-up):**
- `limit = max(ask + 0.0005 * mid, ask + 0.01)` — 5bp above ask, with a 1-cent minimum buffer
- For tickers with mid < $20: use a flat 0.05 cent buffer above ask instead of bp-based
- Round to penny for stocks ≥ $1.00

**Sell limits (authority c — trim):**
- `limit = min(bid - 0.0005 * mid, bid - 0.01)` — 5bp below bid

**Pre-market quote handling:** if `is_open = false` and the quote is from the prior close (timestamp >1h old), use the prior close's bid/ask plus a 10bp buffer (wider, since pre-market spreads are uncertain). Note this in the rationale's 🚫 DISQUALIFIERS line: "Pre-market limit may not fill on open if today's open gaps significantly from yesterday's close."

**Trail stops (authority stop):**
- Use `place_stock_order` with `order_type = "trailing_stop"`, `trail_percent = [tier]`, `time_in_force = "gtc"`
- Tier per the risk-tier reference table

**Time in force:** `day` for buys/sells (a/b/c). `gtc` for trail stops.

If `get_stock_latest_quote` returns stale data (>60s old during market hours, >12h old pre-market) or zero bid/ask: defer the candidate, log "stale quote — could not safely set limit."

---

*End of playbook. The agent should exit after Step 12.*
