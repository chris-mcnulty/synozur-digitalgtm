---
description: Detect sends and replies, fire pre-approved templated follow-ups, decide when to stop. Spawns the Cadence subagent.
---

# /run-cadence

Run the Cadence agent over every in-flight prospect.

## Eligible states

- `draft_pending_approval` — check Outlook sent-items for the matching
  subject; if found, transition to `sent`.
- `sent` — check inbox for a reply on the thread.
- `awaiting_reply` — same as `sent`, plus check whether the next
  templated step is due (`next_step_at` in the past).
- `cadence_step_due` — fire the next templated step (auto-send if every
  guardrail passes; otherwise hold for human review).
- `replied` — classify if not yet classified.

## Steps

1. Check kill switches. Honor `AGENTS_PAUSED`.
2. List candidates from `$PROSPECT_DIR/`.
3. Compute campaign-level reply rate (last 50 sends across all leads
   sharing a `campaign:` tag). If below `MIN_REPLY_RATE_PCT`, auto-pause
   that campaign and skip its `cadence_step_due` leads.
4. Spawn the `cadence` subagent. Sequential.
5. Print a summary:
   - sends detected
   - replies detected (with classification breakdown)
   - templated steps auto-fired
   - templated steps held (with reason)
   - leads moved to dormant
   - campaigns auto-paused

$ARGUMENTS
