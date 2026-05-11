# Midday Trading Playbook — Execute v1

*Paste this into `github.com/Dylanxmt/finance-playbooks/midday-execute-v1.md`. Version: 2026-05-10 (initial — graduated from `midday-dryrun-v1.md` after dry-run validation).*

*This file is fetched at runtime by the Anthropic scheduled trigger. **This playbook executes real orders against the configured Alpaca account.** Verify `ACCOUNT_ID` resolves to a paper account before adopting on the trigger.*

---

## 🟢 EXECUTE MODE — what changed from dry-run

This is the live-action successor to `midday-dryrun-v1.md`. The decision logic (Steps 1-6) is identical; the difference is what happens at Step 7. In dry-run, candidates that pass guardrails became Gmail rationale blocks. Here they become actual `place_stock_order` calls plus trail-stop follow-ups.

**Allowed order tools** (only these — never any others):
- `place_stock_order`
- `cancel_order_by_id` (only to cancel a stale midday-placed limit that timed out)
- `replace_order_by_id` (only for tightening trail stops per authority `c'`)

**Forbidden in midday execute** (explicitly):
- `place_crypto_order`, `place_option_order` — separate playbooks if/when needed
- `close_position`, `close_all_positions` — midday never exits a full position; trail stops own that
- `exercise_options_position`, `do_not_exercise_options_position` — N/A
- `update_account_config` — never modify account settings from an agent

If you catch yourself about to call a forbidden tool, STOP. Finalize the email with what was placed (or attempted) and exit.

---

## 🟡 MIDDAY-SPECIFIC AUTHORITY

Midday runs at **12:00 PM ET** — 2.5 hours into the session. Its job is **reactive**, not expansive.

**Allowed authorities:**
- **(b) Top up** an existing winner on a fresh intraday catalyst
- **(c) Trim** a position that breached the 12% concentration line during the session
- **(c') Tighten a stop** on a position where intraday news has weakened the thesis (replace the trail stop with a tighter %)

**NOT allowed:**
- **(a) Open a new position** — new names go through morning only. Carry-forward to next morning's playbook if compelling.
- Anything touching a ticker already covered by an active morning order — defer.

**Default stance: stand down.** If nothing material has changed since morning, the email is a short monitoring summary and zero orders fire. Most days end here.

---

## Context injected by trigger

- **Mode:** EXECUTE
- **Approval mode:** `{{APPROVAL_MODE}}` — one of `auto` (default) or `manual`. In `manual`, the playbook builds the order plan + Gmail draft but does NOT call any place/replace/cancel tool. Use `manual` for the first 1-2 weeks of execute mode if you want a final human gate, then flip to `auto` once you trust it.
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

## Hard caps (enforced in Step 6)

- Max per single trade: **5% of equity**
- Min cash reserve: **25% of equity** (never breach, even for top-ups)
- Order types: **LIMIT only** — day, never market, never extended hours
- Max trades this session: **2** (midday is reactive, not active)
- Max trail stop replacements (authority c'): **2** per session, separate from the 2-trade cap

---

## Step 0 — Kill Switch Check (two paths, either engaged → standdown)

Two independent kill paths. Check the repo file first (primary), then Gmail label (secondary). Either engaged → standdown. **In execute mode this check is especially critical — past Step 0, the agent has authority to place real orders.**

**Step 0a — Repo kill-switch file (primary path).**
1. Fetch via Bash + curl (10s timeout):
   ```
   curl -s --max-time 10 https://raw.githubusercontent.com/Dylanxmt/finance-playbooks/main/kill-switch/state.txt
   ```
2. If fetch succeeds: parse line 1 (trimmed of whitespace).
   - If line 1 is exactly `PAUSED`: standdown immediately, do NOT proceed to any order tool. Create draft titled `Midday Brief — [Date] — STANDING DOWN (repo kill switch ENGAGED — execute mode aborted)`. Body: include the full file contents so Dylan sees the metadata footer. Add a line "Disengage at: https://github.com/Dylanxmt/finance-playbooks/blob/main/kill-switch/state.txt". Exit.
   - If line 1 is `ARMED` or anything else: proceed to Step 0b.
3. If fetch fails: sleep 5s, retry once.
4. If both fetch attempts fail: set `repo_kill_reachable = false`, log the failure, proceed to Step 0b.

**Step 0b — Gmail label (secondary path).**
5. Search Gmail for `label:TRADING-PAUSED newer_than:2d`.
6. If ANY match found:
   - Standdown, do NOT proceed to any order tool. Create draft titled `Midday Brief — [Date] — STANDING DOWN (Gmail kill label active — execute mode aborted)`.
   - Body: one line stating "Kill switch detected via Gmail label. Remove the `TRADING-PAUSED` label to resume."
   - Exit.
7. If the Gmail search itself errors: set `gmail_kill_reachable = false`, log the failure, continue.

**Step 0c — Fail-closed if both paths unreachable.**
8. If `repo_kill_reachable = false` AND `gmail_kill_reachable = false`: stand down. **Execute mode fails closed harder than dry-run** — without confirmed safety signal, do NOT place orders. Subject `Midday Brief — [Date] — STANDING DOWN (both kill paths unreachable — execute mode fail-closed)`. Body explains both failures so Dylan can investigate before next run.
9. If either path is reachable and reports armed/no-label, proceed to Step 1.

## Step 1 — Health Check (discovery-tolerant + degraded-mode fallback)

Identical to `midday-dryrun-v1.md` Step 1 (Phases A/B/C/D). Do not duplicate logic — fetch and apply the same sequence. Key facts that matter for execute mode:
- If Phase D fires (degraded mode), `alpaca_available = false` — **NO orders may be placed** in degraded mode (order plumbing unreachable). Email reports degraded state and exits without touching any order tools.
- If Phase C account check fails (account blocked) → true standdown, no orders.
- Only proceed past Step 1 with `alpaca_available = true` and `position_source = "alpaca-live"`.

## Step 2 — Parse Morning Brief

Identical to `midday-dryrun-v1.md` Step 2 — extract `morning_proposed_tickers[]`, `morning_rejected_tickers[]`, `morning_flagged_tickers[]`, and `active_signals[]` from this morning's email.

**Execute-mode addition:** also extract any `client_order_id` values referenced in the morning email. The morning execute playbook (when it exists) writes its placed-order IDs into the email under a "💼 ORDERS PLACED" section. Capture `morning_placed_order_ids[]` so midday can reconcile against today's actual fills in Step 3.

## Step 3 — Portfolio Snapshot (Intraday)

Same data calls as `midday-dryrun-v1.md` Step 3, with one execute-mode addition:

**Reconcile against morning's placed orders:**
- For each `client_order_id` in `morning_placed_order_ids[]`, call `get_order_by_client_id`.
- Record fill status: `filled` / `partial` / `cancelled` / `still_working`.
- For partial fills, record `filled_qty` separately — Step 6's cash math uses actual filled qty, not the originally proposed qty.
- For still-working morning limits, treat their reserved cash as committed (subtract from available cash before midday's own cash check).

## Step 4 — Midday Scan (reactive only)

Same triggers and same default-stand-down posture as `midday-dryrun-v1.md` Step 4. No execute-mode behavior change here — the scan produces candidate authorities, not orders.

## Step 5 — Quick Market Read (compact)

Same as `midday-dryrun-v1.md` Step 5.

## Step 6 — Guardrail Filter

Same checks as `midday-dryrun-v1.md` Step 6. **Execute-mode hardening: each guardrail check uses LIVE values, not estimates.**

For each surviving candidate compute the actual cost using the latest NBBO (see Step 7a for limit-price math). Re-run the cash floor check using `cash − sum(working morning limits' reserved cash) − sum(midday candidate costs)`. If the cumulative result < 25% equity, drop the lowest-conviction candidate and re-check. Repeat until set passes or empty.

Log every drop with the specific cash math under "🗑️ CONSIDERED BUT REJECTED" in the email.

## Step 7a — Pre-Execution Sanity

Run these checks ONCE just before the first order call. If any fails, abort the entire order batch (skip 7b/7c/7d) and produce a "near-miss" email noting what failed.

For each surviving candidate:
1. Fetch `get_stock_latest_quote` for the ticker. Confirm the quote is fresh (`timestamp` within last 60s).
2. Confirm bid/ask spread is sane: `(ask − bid) / mid < 0.5%` for liquid names, `< 1.0%` for small-caps. Wider than that → defer this candidate (note as "spread too wide for safe limit").
3. Confirm `get_clock` still reports market open with `is_open = true` and >30 min until close. (Skip step 7 entirely if <30 min remain — let EOD handle.)
4. For trims (authority c): confirm the position still qualifies at >12% of equity using current price. The intraday move that triggered the trim might have reversed since Step 4.
5. For top-ups (authority b): confirm the position is still >5% above the morning HWM with the catalyst from Step 4 still active in current news.

If `APPROVAL_MODE = manual`: skip Step 7b/7c — write the order plan into the email under "💼 PROPOSED FOR YOUR APPROVAL" and exit. Dylan executes manually if he agrees.

## Step 7b — Order Placement

Place orders one at a time in this priority order (NEVER parallelize order calls — race conditions on cash math):

1. **Tightenings (authority c')** — fastest, no cash impact. Use `replace_order_by_id` on the existing trail stop. Same `client_order_id` reuse is forbidden — generate a new ID per Appendix B2.
2. **Trims (authority c)** — frees cash. Use `place_stock_order` (sell, limit, day). After fill, cancel + replace the existing trail stop with a smaller-qty version (same trail %).
3. **Top-ups (authority b)** — uses cash. Use `place_stock_order` (buy, limit, day).

For each order:
- Compute limit price (Appendix E).
- Generate `client_order_id` per Appendix B2: `midday-{YYYY-MM-DD}-{ticker}-{authority}-{seq}`. Sequence is per-(date+authority+ticker), starts at `001`.
- Call `place_stock_order` (or `replace_order_by_id` for c').
- Capture the returned `id` (Alpaca order ID), `client_order_id`, submitted timestamp.
- If the call errors: log the error with the candidate's full context. Skip this candidate, do NOT retry. Continue to the next candidate.

Hold all submitted orders as `submitted_orders[]` for Step 7d.

## Step 7c — Trail Stop Placement (top-ups only)

For each top-up that submitted successfully in 7b, queue a corresponding trail stop. **Do not place the trail until the parent buy is confirmed filled** — Step 7d handles the timing.

**Sizing rule (separate-trail policy):** the new trail stop covers ONLY the newly purchased shares, not the combined position. The original position keeps its existing trail at its existing HWM. This preserves any unrealized-gain protection that the original trail has already accrued.

Example: existing 100-share NVDA position with a 5% trail at HWM $620 (stop $589). Top-up adds 30 shares at $635. After top-up:
- Original 100-share trail: untouched. Still HWM $620, stop $589.
- New 30-share trail: HWM $635, stop $603.25. Placed when the top-up fills.

This intentionally creates two trails per ticker until the original trail eventually exits. EOD's reconciliation handles the dual-trail attribution by `client_order_id` prefix.

`client_order_id` for the new trail: `midday-{YYYY-MM-DD}-{ticker}-stop-{seq}`. The trail-stop authority label is `stop`, not `topup`.

## Step 7d — Fill Verification

For each entry in `submitted_orders[]`, poll `get_order_by_id` until terminal:

- Poll cadence: every 30 seconds.
- Timeout: 5 minutes from submission.
- Terminal states: `filled`, `partially_filled` (after expiration), `canceled`, `expired`, `rejected`, `replaced`.

On terminal:
- `filled`: record `filled_avg_price`, `filled_qty`, `filled_at`. If this was a top-up parent order, IMMEDIATELY place its corresponding trail stop from 7c (using `filled_qty`, not the originally submitted qty — partial fills cascade).
- `partially_filled` after timeout: cancel via `cancel_order_by_id`, record what filled. If a top-up partially filled, place trail for the actually-filled qty (zero qty → skip trail).
- `canceled` / `expired` / `rejected` / `replaced`: log the reason. Do NOT retry.
- Still working at 5-min timeout: cancel via `cancel_order_by_id`. Log as "timed out — limit too far from NBBO or thin liquidity."

Verify `get_account_info` after the batch: confirm `cash >= equity * 0.25` (the 25% floor). If somehow breached (shouldn't happen if Step 7a math was right), log `CASH_FLOOR_BREACH` to the email loud red — this is a contract violation worth investigating.

## Step 8 — Email Assembly

**Subject format:**
- Monitoring (no triggers fired): `Midday Brief — [Date] — Monitoring, no orders placed`
- Orders executed: `Midday Brief — [Date] — [N] order(s) placed, [M] filled`
- Manual approval mode: `Midday Brief — [Date] — [N] proposed for approval`
- Degraded mode (Phase D): `Midday Brief — [Date] — DEGRADED ([failure_class])`
- Standdown: `Midday Brief — [Date] — Standing down ([brief reason])`

**Body sections** (exact order, omit empty sections):

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
MIDDAY BRIEF — [Date]
Mode: EXECUTE | Authority: reactive only (top-up / trim / tighten)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📊 ACCOUNT
Equity: $X | Cash: $Y (Z%) | Invested: $A (B%)
Intraday: $C (D%) | Since inception: $E (F%)

🔄 SESSION SO FAR
- Morning fills today: [list of morning client_order_ids that filled, with avg price]
- Morning still-working: [list of morning limits still on the book]
- Morning cancelled: [if any]
- Trail-stop fires today: [any]

🌍 TAPE READ (compact)
[same as dry-run Step 5 output]

🧭 ACTIVE SIGNALS (from morning's carry-forward)
[same as dry-run]

📁 POSITION CHECK
[same table as dry-run]

⚡ TRIGGER(S) FIRED
- [Step 4 trigger 1-5 that fired, ticker, specific change]

💼 ORDERS PLACED ([N])
[For each entry in submitted_orders[], render a card with:
- Ticker, side, authority, qty, limit price, client_order_id
- Status: filled @ $X.XX (slippage vs limit: $0.XX) / partial / cancelled / timed out / rejected
- Linked trail-stop client_order_id (for filled top-ups)
- Rationale: signal, thesis, size justification (90-110 words, same format as dry-run Step 9)]

🛑 TRAIL STOPS PLACED OR ADJUSTED ([M])
[For each trail action — placement of new trail for top-up, or replacement for tightening:
- Ticker, qty, trail %, current stop estimate, client_order_id, parent order client_order_id (if linked)]

🗑️ CANDIDATES NOT EXECUTED
[Any candidate dropped in Step 6 (guardrails), 7a (pre-execution sanity), 7b (submission error), or 7d (fill failure). Format: ticker — reason — specific failure point.]

📈 PORTFOLIO IMPACT (post-execution)
Cash: $X (was $Y at midday open, $Z at AM open)
Invested: $A (was $B / $C)
Position count change: N → M

━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ EXECUTE MODE — orders above are LIVE on Alpaca {{ACCOUNT_ID}}
This is paper trading. Verify trail stops survived to close at EOD report.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

For monitoring-only days, omit the ⚡/💼/🛑 sections and add a `💤 NO ACTION TAKEN — REASONING` block (same as dry-run).

## Step 9 — Create Gmail Draft (dual-format)

Same procedure as `midday-dryrun-v1.md` Step 9 — plaintext `body` + styled `htmlBody`, post-create verification via `list_drafts`, retry on silent connector failure, exit non-zero on verification failure.

The HTML rendering of "ORDERS PLACED" cards should use the same yellow-amber rationale-card style from morning playbook Appendix D, but with a **status badge** in the top-right corner of each card:
- `filled`: green badge with fill price
- `partial`: amber badge with filled qty / submitted qty
- `cancelled` / `timed out`: gray badge
- `rejected`: red badge

## Step 10 — Completion

Output a single status line:
```
Midday execute drafted | Orders placed: [N] | Filled: [M] | Equity: $X | Mode: EXECUTE | Status: OK
```

Exit.

---

## Appendix A — Standdown vs Degraded vs Monitoring vs Executed

Four terminal states (one more than dry-run because executed-with-zero-fills is its own state):

**Monitoring-only**: Step 1 succeeded, no Step 4 trigger fired. Subject: "Monitoring, no orders placed." Happy path.

**Executed**: Step 4 fired, candidates survived guardrails, ≥1 order submitted. Subject: "[N] order(s) placed, [M] filled." Could include partial fills, rejections, timeouts — they all show up in the email under their proper sections.

**Degraded**: Step 1 Phase D fired (Alpaca discovery or probe failed but EOD-fallback worked). NO orders placed. Subject includes "DEGRADED."

**True standdown**: kill switch / account blocked / market closed / Phase D also failed / unrecoverable error. NO orders placed. Subject includes "STANDING DOWN."

## Appendix B — Safety Rails Specific to Midday Execute

- **No closing full positions.** Trail stops own exits. Midday can trim or tighten, never close.
- **No re-entering stopped-out names.** If a position stopped out earlier today, do not propose re-entry.
- **No chasing pre-market gappers fading intraday.** That's a morning decision, not midday.
- **If morning stood down**, midday inherits the standdown bias — be extra conservative.
- **Hard stop at 2 reactive trades per session** (separate from the 2 c'-tightening cap).
- **No order placement in the last 30 min before close.** Step 7a aborts if `get_clock` shows <30 min remaining. Let EOD handle late-session decisions.
- **No order placement when `cash < 26% equity` pre-check** (1pp safety margin above the 25% floor — accounts for slippage).
- **Cancel any midday limit that hasn't filled in 5 min.** A stale midday limit on the book overnight would distort tomorrow morning's plan.

## Appendix B2 — `client_order_id` contract

Every `place_stock_order` and `replace_order_by_id` call MUST set `client_order_id` using:

```
midday-{YYYY-MM-DD}-{ticker}-{authority}-{seq}
```

- `{authority}` ∈ {`topup`, `trim`, `tighten`, `stop`}
  - `topup`: parent buy for authority (b)
  - `trim`: parent sell for authority (c)
  - `tighten`: trail-stop replace for authority (c')
  - `stop`: new trail stop placed after a top-up fill (linked to the topup parent)
- `{seq}` = 3-digit zero-padded, per (date + authority + ticker), starts at `001`

Examples:
- `midday-2026-05-15-NVDA-topup-001` — top-up buy
- `midday-2026-05-15-NVDA-stop-001` — trail stop for the topped-up shares
- `midday-2026-05-15-XLK-trim-001` — trim sell
- `midday-2026-05-15-XLK-stop-001` — replacement trail after the trim
- `midday-2026-05-15-FCX-tighten-001` — tighten existing trail (replace)

EOD reconciliation uses the prefix to attribute each fill to the correct agent + authority. NEVER re-use a `client_order_id` — Alpaca will reject the second call.

## Appendix C — Parsing the Morning Email

Same as `midday-dryrun-v1.md` Appendix C plus one addition: extract `morning_placed_order_ids[]` from the morning email's "💼 ORDERS PLACED" section (when it exists — only morning-execute writes this section; morning-dryrun writes "TRADES PROPOSED" without IDs).

## Appendix D — HTML Visual Conventions

Same as `midday-dryrun-v1.md` Appendix D + the order-status badge convention from Step 9.

## Appendix E — Limit Price Calculation

For each order, fetch `get_stock_latest_quote` immediately before submission (Step 7a got a fresh quote — reuse if <30s old, else re-fetch).

**Buy limits (authority b — top-up):**
- `limit = max(ask + 0.0005 * mid, ask + 0.01)` — 5bp above ask, with a 1-cent minimum buffer for low-priced stocks
- For tickers with mid < $20: use a flat 0.05 cent buffer above ask instead of bp-based
- Round to penny for stocks ≥ $1.00, tenth-of-penny for stocks < $1.00 (we won't hit this — no penny stocks in the universe — but spec it anyway)

**Sell limits (authority c — trim):**
- `limit = min(bid - 0.0005 * mid, bid - 0.01)` — 5bp below bid

**Trail stops (authority stop):**
- Use `place_stock_order` with `order_type = "trailing_stop"`, `trail_percent = [tier]`, `time_in_force = "gtc"`
- Tier per the risk-tier reference table

**Trail-stop tightening (authority c'):**
- Use `replace_order_by_id` on the existing trail. New `trail_percent` = old percent / 1.5 (e.g., 4% → 2.67%) UNLESS the new value would be < 2%, in which case use 2% (Appendix B says no tightening to within 1% of current; 2% gives a 1pp safety margin from same-day exit).

**Time in force:** `day` for buys/sells (b/c). `gtc` for trail stops (stop/c').

If `get_stock_latest_quote` returns stale data (`timestamp` >60s old) or zero bid/ask: defer the candidate, log "stale quote — could not safely set limit."

---

*End of playbook. The agent should exit after Step 10.*
