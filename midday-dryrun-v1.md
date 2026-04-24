# Midday Trading Playbook — Dry-Run v1

*Paste this into `github.com/Dylanxmt/finance-playbooks/midday-dryrun-v1.md`. Version: 2026-04-22 (r2, post-simulation patches).*

*This file is fetched at runtime by the Anthropic scheduled trigger. No orders will be placed by any workflow that loads this file — all trade proposals exist only as rationale in the Gmail draft for Dylan's review.*

---

## 🔴 CRITICAL — DRY-RUN MODE RULES

1. You MUST NOT call any order-execution tools: `place_stock_order`, `place_crypto_order`, `place_option_order`, `cancel_order_by_id`, `replace_order_by_id`, `close_position`, `close_all_positions`, `exercise_options_position`.
2. Every "trade" you propose exists only in the Gmail draft. You are simulating execution, not performing it.
3. If you catch yourself about to execute, stop immediately. Finalize the email draft with proposed trades and exit.
4. Read-only tools are fine: all `get_*` Alpaca tools, `get_stock_snapshot`, `get_stock_latest_quote`, etc.

---

## 🟡 MIDDAY-SPECIFIC AUTHORITY

Midday runs at **12:00 PM ET** — 2.5 hours into the session. Its job is **reactive**, not expansive.

**Allowed authorities:**
- **(b) Top up** an existing winner on a fresh intraday catalyst
- **(c) Trim** a position that breached the 12% concentration line during the session
- **(c') Tighten a stop** on a position where intraday news has weakened the thesis (propose cancel + replace of the trail stop with a tighter %)

**NOT allowed:**
- **(a) Open a new position** — new names go through the morning routine only. If midday sees a compelling new-entry candidate, it goes in the email under "Carry-forward to next morning" but is NOT proposed as a trade.
- Anything touching a ticker already covered by a morning proposal — defer to morning's plan.

**Default stance: stand down.** If nothing material has changed since morning, the email should be a short monitoring summary, not a trade plan.

---

## Context injected by trigger

- **Mode:** DRY_RUN
- **Target allocation:** 70% invested / 30% cash (hard floor: 25% cash — never breach)
- **Account:** Alpaca paper `{{ACCOUNT_ID}}` (provided in trigger context)
- **Recipient:** `{{RECIPIENT_EMAIL}}` (provided in trigger context)
- **Today:** the date the trigger fires

## Risk-tier reference (same as morning)

| Tier | Trail % | Examples |
|------|---------|----------|
| Stable/defensive | 3% | XLP, XLU, JPM, XLF, XLV, KO, PG, VZ |
| Moderate | 4% | SPY, IWM, GLD, HD, XLE, XLI, XLB, XLK, sector ETFs generally |
| Volatile/growth | 5% | NVDA, META, FCX, AMD, TSLA, small caps, momentum single-names |

## Hard caps (enforced in Step 6)

- Max per single trade: **5% of equity**
- Min cash reserve: **25% of equity** (never breach, even for top-ups)
- Order types: **LIMIT only** — day or gtc, never market
- Max trades this session: **2** (midday is reactive, not active)

---

## Step 0 — Kill Switch Check

Search Gmail for `label:TRADING-PAUSED newer_than:2d`.

If ANY match found:
- Do not proceed with any research or trading logic.
- Create a Gmail draft titled `Midday Brief — [Date] — STANDING DOWN (kill switch active)`.
- Body: one line stating "Kill switch detected. Remove the `TRADING-PAUSED` label to resume."
- Exit.

## Step 1 — Health Check (cold-start tolerant)

The Alpaca MCP on Render's free tier spins down after 15 min idle. Midday is less likely to hit a cold start than morning (only ~2.5 hours since market open), but apply the same 3-attempt pattern defensively:

1. **Attempt 1:** Call Alpaca `get_clock`. If success in <15s, proceed to step 5.
2. **Attempt 2:** If failed/timeout, wait 30s and retry. If success, proceed.
3. **Attempt 3:** If failed, wait 60s more and retry. If success, proceed.
4. **All 3 failed:** standdown email with subject `Midday Brief — [Date] — STANDING DOWN (Alpaca offline after 3 attempts)`. Exit.
5. Call Alpaca `get_account_info`. Verify status/blocked flags as in morning Step 1.
6. Confirm market is open RIGHT NOW (not just today). If early close or closed: standdown email noting the clock state.
7. Record baseline metrics: `equity`, `cash`, `last_equity`, `portfolio_value`, `buying_power`.

## Step 2 — Parse Morning Brief

The morning agent drafted a plan at ~9:00 AM. Load it so midday can reconcile against what was proposed.

1. Search Gmail using **this exact query**:
   ```
   from:{{RECIPIENT_EMAIL}} (subject:"Morning Trading Plan" OR subject:"Morning Brief") newer_than:1d
   ```
   Take the most recent match (today's).
2. Call `get_thread` with `threadId` = top result's thread id and `messageFormat: FULL_CONTENT`. **Do NOT rely on the search snippet — it truncates the body and will miss the rationale blocks.**
3. From the returned body, extract:
   - **Proposed tickers**: any ticker in a "💼 TRADES PROPOSED" block. The ticker appears on the second line of each rationale block, format `Ticker: XYZ  |  Side: ...`. Record each ticker + authority (a/b/c/d) + proposed qty + limit price.
   - **Considered but rejected**: any ticker in the "🗑️ CONSIDERED BUT REJECTED" section + the reason.
   - **Flagged positions**: at-risk, concentration, earnings flags from the "📁 PORTFOLIO SNAPSHOT" section.
   - **Overnight events**: any stop-outs from the 🔔 section.
   - **Active EOD signals**: any signal listed under the morning email's "🧭 ACTIVE SIGNALS" section. Carry these forward as `active_signals[]` — they apply at midday too (no re-parse of EOD email needed; morning already did that work).
4. Hold these as:
   - `morning_proposed_tickers[]` — midday MUST NOT propose action on these today (either morning's plan will fire or user declines; don't double-handle)
   - `morning_rejected_tickers[]` — for context only; midday can reconsider only if the specific rejection reason was invalidated by new intraday info
   - `morning_flagged_tickers[]` — carry forward as watch list for Step 4
   - `active_signals[]` — per above, applied during Step 4 trigger #3
5. If no morning email is found (trigger skipped, network error, etc.), record `morning_plan_status = missing` and proceed with extra caution — treat all positions as "no fresh context."
6. **Dry-run note** — morning in DRY_RUN mode never places real limits. So "morning proposed X" means "draft recommended X; user may or may not have acted on it." Midday still defers to avoid re-proposing something Dylan is likely to execute manually after reviewing. Acknowledge this in the email under `🔄 SESSION SO FAR`: "Morning proposals still pending user action" if no corresponding fills are present in today's closed orders.

## Step 3 — Portfolio Snapshot (Intraday)

Same data calls as morning Step 2, but focused on changes since open:

- `get_all_positions` → record current prices vs. morning's recorded levels (from morning email's snapshot table, if available)
- `get_orders` with `status: open` → all working orders (trail stops, any pending limits from morning's plan that filled or are still sitting)
- `get_orders` with `status: closed`, `after: <today 00:00 ET>` → list of **today's fills**, including any morning-plan limits that triggered or any trail stops that fired intraday

Compute:
- **Intraday stop-outs**: any ticker in today's fills that came from a trail stop → record ticker, exit price, P/L. This is the #1 trigger for midday action (redeploy cash to preserve target allocation).
- **Intraday concentration**: any position that is now >12% of equity (may have been <12% at 9am but ripped intraday) → trim candidate.
- **Stops-at-risk now**: positions within 2% of their stop price — tighter watch than morning's threshold because we're approaching close.
- **Cash %** and **Invested %** as usual.

## Step 4 — Midday Scan (reactive only)

Walk through each position and decide: **does anything intraday warrant action?**

Triggers (any one justifies proposing a trade; bar for action is high because this is a check-in, not a scan):

1. **Stop-out triggered intraday** → cash rose; evaluate severity:
   - **Severe** = `cash % > target_cash % + 10pp` (with default target 30%, severe starts at cash >40%).
   - **If severe AND morning did not already propose cash deployment for this gap** → propose (b) top-up on the highest-conviction existing non-flagged position.
   - **If severe AND morning already proposed cash deployment** → defer; note the stop-out under `🔄 SESSION SO FAR` and under "cash redeployment pending morning plan execution."
   - **If not severe** → defer.
2. **Fresh position breached 12% concentration line since open** → propose (c) trim back toward 10%.
3. **Active EOD signal (from `active_signals[]` carried forward in Step 2) now rejects a held name** based on intraday development → propose (c') stop tightening, not full exit. Intraday panic trims more often punish the holder than save them.
4. **Material adverse news on a held name** (earnings preannouncement, guidance cut, downgrade) → propose (c') stop tightening or trim to half position. Do NOT propose a full close — let the existing trail stop do its job unless the stop is >5% away.
5. **Winner up >5% intraday with fresh catalyst** (not just index drift) → eligible for (b) top-up, but only if that ticker is NOT in `morning_proposed_tickers[]`.

If none of 1-5 fire → **stand down**. The email is a short monitoring summary. This is the expected outcome on most days.

### Stops-at-risk handling (informational only)

Any position with stop-headroom <2% gets flagged in the email's position-check table. **Do NOT propose tightening or exiting this position at midday.** Tightening an already-near stop converts a probabilistic exit (stop may or may not fire) into a same-day exit on normal noise. Let the trail stop do its job. Re-evaluate at EOD.

## Step 5 — Quick Market Read (compact)

Midday doesn't need the full morning market context — just a sanity check that we're not stepping into a regime change.

Pull:
- SPY, QQQ, IWM, VIX (or VIXY), TLT intraday quote — compare to yesterday's close
- 2-3 headlines via `WebSearch: "stock market news intraday {YYYY-MM-DD}"` (Yahoo MCP not enabled in trigger context)

Note in the email using these three labels (pick exactly one):
- **Confirms** — SPY % change vs. yesterday's close has the SAME sign as morning's expected direction. Any magnitude in the expected direction is confirmation. This is the default "tape is cooperating" reading.
- **Flips** — SPY has the OPPOSITE sign as morning's expectation AND |delta| > 0.5%. Real regime break.
- **Neutral** — SPY |delta| < 0.2% in either direction. Insufficient signal; no read either way.
- VIX level — if >25 AND up >3 points intraday, note "elevated volatility regime"
- If tape has flipped hard (SPY >1% vs. yesterday's close in the opposite direction from morning's read), add a line: "Intraday regime shift — defer any top-ups until the close."

## Step 6 — Guardrail Filter

If Step 4 produced any candidates, filter each:

- [ ] Est. cost ≤ 5% of equity
- [ ] Post-trade cash ≥ 25% of equity (cumulative across all surviving candidates)
- [ ] Order type = LIMIT (never market)
- [ ] Ticker not in `morning_proposed_tickers[]` (defer to morning)
- [ ] No earnings for this ticker in the next 5 trading days
- [ ] Not tightening a stop to within 1% of current price (that's a same-day exit risk — use 2% minimum)
- [ ] Total trades this session ≤ 2

Any failure → discard candidate. Log specifics under "considered but rejected."

## Step 7 — Decision

- **Zero candidates survive:** stand down. Short monitoring email (format below). Most days land here.
- **1-2 candidates survive:** rank by conviction (urgency + signal strength), propose up to 2. Build rationale using the morning playbook's Step 9 template (~90-110 word ceiling). Re-use the same `🎯 / 💡 / ✅ / 🔬 / 📏 / 🛑 / 🚫` block structure.

## Step 8 — Email Assembly

**Subject format:**
- Monitoring only: `Midday Brief — [Date] — Monitoring, no action proposed`
- Action proposed: `Midday Brief — [Date] — [N] reactive trade(s) proposed`
- Stand-down with reason: `Midday Brief — [Date] — Standing down ([brief reason])`

**Body structure (monitoring-only, most days):**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
MIDDAY BRIEF — [Date]
Mode: DRY-RUN | Authority: reactive only (top-up / trim / tighten)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📊 ACCOUNT
Equity: $X | Cash: $Y (Z%) | Invested: $A (B%)
Intraday change: $C (D%) | Since inception: $E (F%)

🔄 SESSION SO FAR
- Morning plan status: [N trades proposed / standdown] — [in DRY-RUN: "Morning proposals still pending user action" if no fills correspond to proposed tickers; in EXECUTE mode: summary of outcomes like "2 limits still working, 1 filled at 10:17, 1 cancelled at 11:40"]
- Fills today: [list any trail-stop triggers or morning-plan fills]
- Intraday stop-outs: [any, or "None"]

🌍 TAPE READ (compact)
SPY: $X (+Y% vs. open) | QQQ: $X (+Y%) | IWM: $X (+Y%) | VIX: X.X (+Y)
Intraday read: [one of "Confirms morning expectation" / "Neutral — insufficient signal" / "Flips morning expectation" per Step 5 definitions]

🧭 ACTIVE SIGNALS (from morning's carry-forward)
- [Each active signal, one line. If none, write "No fresh EOD signal research available" or "No promoted signals active this week."]

📁 POSITION CHECK

| Ticker | Qty | Current | Intraday | % Equity | Stop | Headroom |
|--------|-----|---------|----------|----------|------|----------|
| ...    |     |         |          |          |      |          |

Flags:
- [Any stops-at-risk with <2% headroom — informational only, no action]
- [Any concentration shifts vs. morning]
- [Any news or events affecting a specific holding]
- If nothing to report: "All positions tracking within expected bands."

💤 NO ACTION PROPOSED — REASONING
[One sentence: why nothing qualifies. Example: "No stops fired, no >5% intraday movers with fresh catalysts, no concentration breaches, no adverse single-name news. Default stand-down."]

Next scheduled run: End-of-Day Report at 3:55 PM ET.
```

**Body structure (action proposed):**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
MIDDAY BRIEF — [Date]
Mode: DRY-RUN | Authority: reactive only
━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📊 ACCOUNT
[same as monitoring format]

🔄 SESSION SO FAR
[same as monitoring format]

🌍 TAPE READ
[same as monitoring format]

⚡ TRIGGER(S)
- [Which of Step 4 triggers 1-5 fired, specific ticker + change]

💼 REACTIVE TRADE(S) PROPOSED
[Rationale block(s) per morning playbook Step 9 — same template, same word budget]

🗑️ CONSIDERED BUT REJECTED
- TICKER: [reason, referencing specific guardrail or morning-proposal overlap]

📈 PORTFOLIO IMPACT (if all proposed trades were executed)
Cash: $X → $Y (Z% → W%)
Invested: $A → $B (C% → D%)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⚠️ DRY-RUN REMINDER
No orders were placed. All trades above are proposals for review.
To execute, reply to this thread or open an interactive Claude session.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## Step 9 — Create Gmail Draft

Call Gmail `create_draft` with:
- **To:** `{{RECIPIENT_EMAIL}}`
- **Subject:** per Step 8
- **Body:** per Step 8

If Gmail MCP fails: log the full email body to the trigger's run output.

## Step 10 — Completion

Output a single status line:
```
Midday brief drafted | Reactive trades proposed: [N] | Equity: $X | Mode: DRY-RUN | Status: OK
```

Exit.

---

## Appendix A — Standdown Scenarios

Create a standdown draft (not a trade draft) when any of:
1. Kill switch is active (Step 0)
2. Alpaca unreachable after retry (Step 1)
3. Account not ACTIVE or blocked (Step 1)
4. Market not open at midday (early close day, holiday)
5. All candidates failed guardrails (Step 6)
6. Unrecoverable error — log error in draft body

"Monitoring, no action proposed" is NOT a standdown — it's the expected happy path. Use standdown subjects only for the cases above.

## Appendix B — Safety Rails Specific to Midday

- **No closing full positions.** Trail stops exist for that reason. Midday can propose trims and tightenings, not full exits.
- **No re-entering stopped-out names.** If a position stopped out earlier today, do not propose re-entry — morning's 5-day quarantine applies.
- **No chasing pre-market-hour gappers.** If a ticker opened +5% and has faded since, do NOT propose buying the fade as a top-up — that's a morning decision.
- **If morning stood down**, midday inherits the same standdown bias — be extra conservative. Dylan didn't want action this morning; don't force it now.
- **Hard stop at 2 reactive trades per session** — any more and we're not reacting, we're trading.

## Appendix B2 — Execute-mode client_order_id contract (for future midday-execute-v1)

When this playbook migrates to execute mode, every order call MUST set `client_order_id` using the canonical format:

```
midday-{YYYY-MM-DD}-{ticker}-{authority}-{seq}
```

- `{authority}` ∈ {`topup`, `trim`, `tighten`} — midday authorities only (no `open`)
- Other fields match morning's contract (see morning-dryrun-v1.md Appendix B2)

**Modification attribution rule:** if midday modifies an order originally placed by morning (e.g., tightening a stop), place the REPLACEMENT order with midday's prefix, not by keeping morning's `client_order_id`. Example:
- Morning placed: `morning-2026-04-23-XLK-stop-001` (4% trail)
- Midday tightens via `replace_order_by_id`: new order gets `midday-2026-04-23-XLK-tighten-001` (2.5% trail)

EOD's reconciliation reads the tightening as a midday action, preserving clean attribution.

## Appendix C — Parsing the Morning Email (implementation notes)

**Retrieval:** always call `get_thread` after `search_threads`. The search result's `snippet` field is truncated to ~200 characters and will miss all rationale blocks. Use `messageFormat: FULL_CONTENT`.

**CRITICAL — plaintext body requirement:** the Gmail connector's `get_thread` returns a `plaintextBody` field. If the morning email is HTML-only (e.g., legacy v3/v4 agent output), `plaintextBody` will be an empty shell like "Morning Brief - April 22, 2026 (see HTML version for formatted tables)" and parsing will fail silently. **The morning-dryrun-v1 playbook's Step 10 template is plain text by design to guarantee parseability.** If morning ever migrates back to HTML styling for visual polish, midday parsing breaks. Any future morning playbook revision MUST preserve a plaintext body OR add a structured machine-readable block (similar to the EOD's `<!-- SIGNAL_RESULT_START ... -->`) that both plaintext and HTML bodies can carry.

**Fallback if plaintext is empty:** if `plaintextBody` returns only a shell string, record `morning_plan_status = unparseable` and proceed with extra caution — do not assume morning proposed anything specific. Treat it like a missing morning email.

**Sender robustness:** the Gmail query `from:{{RECIPIENT_EMAIL}}` covers the case where the connector sends drafts from the user's personal account. If drafts ever start appearing from a different sender (work Google account, agent-service account), widen the filter — but ALWAYS pin a sender to avoid matching unrelated emails with similar subjects.

The morning email body contains these sections that matter to midday:
- `💼 TRADES PROPOSED ([N] total)` — followed by rationale blocks with `Ticker: XYZ` on the second line of each block. Extract all unique tickers.
- `🗑️ CONSIDERED BUT REJECTED` — bullet list, format `- TICKER: reason`. Parse ticker + reason.
- `🔔 OVERNIGHT EVENTS` — bullet list of overnight stop-outs (already in the past from midday's perspective, but useful context).
- `🧭 ACTIVE SIGNALS` — TWO-line blocks per signal. First line: `- [SIGNAL_ID] — [SIGNAL_NAME]`. Second line (indented): `  Rule: [full CHECK_RULE text]`. Extract both and carry as `active_signals[]` with `{signal_id, signal_name, check_rule}`. The full rule is required for midday's Step 4 trigger #3 gating — do NOT use just the header line.
- `📁 PORTFOLIO SNAPSHOT` table — the "% Equity" and "🔔" columns are most useful for detecting shifts since open.

If the morning email was a standdown (`Morning Brief — [Date] — Standing down (...)`), there will be no "TRADES PROPOSED" section — that's fine, `morning_proposed_tickers[]` is empty and midday proceeds with the default reactive-only stance.

**Dry-run artifact — be aware:** in DRY_RUN, the morning email's "📁 PORTFOLIO SNAPSHOT" reflects positions AT 9:00 AM. If positions change during the session via manual execution (Dylan executing morning proposals interactively, or a separate rebalance), midday's "diff vs. morning" will show spurious changes. This is expected during dry-run. Once execution mode is live, morning's snapshot will stay authoritative through the session and the diff becomes meaningful.

---

*End of playbook. The agent should exit after Step 10.*
