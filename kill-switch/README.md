# Kill Switch

The agents (morning, midday) check two independent kill paths at Step 0 before doing anything else. Either path engaged → standdown.

## Path 1 (primary): this repo file

`kill-switch/state.txt` — line 1 is the state.

- `ARMED` — trading allowed
- `PAUSED` — kill switch engaged, agents standdown

Fetched at runtime via:

```
https://raw.githubusercontent.com/Dylanxmt/finance-playbooks/main/kill-switch/state.txt
```

**To engage:**
1. Edit `state.txt`, change line 1 to `PAUSED`.
2. Update the metadata footer with date / reason.
3. Commit + push.
4. Next scheduled agent run (morning 9 AM, midday 12 PM) will see `PAUSED` and standdown.

**To disengage:**
1. Edit `state.txt`, change line 1 back to `ARMED`.
2. Update the metadata footer.
3. Commit + push.

You can do all of this from the GitHub mobile app — no laptop required.

## Path 2 (secondary): Gmail label

Apply the `TRADING-PAUSED` label to any Gmail thread newer than 2 days. Agents query
`label:TRADING-PAUSED newer_than:2d` and standdown on any match.

Slower to engage (requires Gmail UI), and depends on the Gmail MCP connector being healthy.
Kept as a fallback so the system has redundancy.

## Why two paths

The 2026-05-01 incident showed the Gmail connector can return fake-success silently. If the
connector breaks and the agent can't read the kill label, the file path is the safety net.
Conversely, if GitHub has a hiccup and the file fetch fails, the Gmail label is the fallback.

## Failure handling

If the agent can't fetch this file (network error, GitHub down), it logs the failure prominently
in the email body and falls through to the Gmail label check. If BOTH paths are unreachable,
the agent standsdown — explicit fail-closed posture only when both checks fail.

## What's NOT a kill switch

- Revoking the Alpaca API key. Nuclear, also kills Dylan's manual trading. Reserve for true emergencies.
- Deleting the trigger. Heavyweight — also loses the schedule config and run history.
- Pausing the trigger via claude.ai/code/scheduled UI. Effective but requires laptop + auth.

The kill switch above is for fast, reversible, mobile-friendly standdown.
