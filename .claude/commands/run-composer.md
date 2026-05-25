---
description: Draft outbound for every prospect in state=researched or state=replied. Spawns the Composer subagent on each.
---

# /run-composer

Run the Composer against every prospect file in `$PROSPECT_DIR/` whose
YAML frontmatter has `state: researched` or `state: replied`.

## Pre-flight check

Before spawning anything, read `skills/outbound-voice/voice-dna.md`. If
it still contains the marker `PLACEHOLDER` anywhere in the file, stop
and tell the operator to:

1. Curate 20 best-performing Synozur sent messages into
   `skills/outbound-voice/corpus/` (gitignored).
2. Run the extraction prompt at
   `skills/outbound-voice/voice-dna-extract.md`.
3. Commit the resulting `voice-dna.md`.

Do not draft anything until Voice DNA is real.

## Steps

1. Read `$PROSPECT_DIR/`. List candidates (`state: researched` or `replied`,
   `do_not_contact: false`).
2. Check `DAILY_SEND_CAP_EMAIL` against the count of leads already in
   `state: draft_pending_approval` + `state: sent` today. Stop drafting
   once the cap is reached.
3. Check kill switches. Honor `AGENTS_PAUSED`.
4. For each candidate, spawn the `composer` subagent. Sequential.
5. After each, log: prospect slug, draft subject, voice-cues applied,
   compliance-checks status.
6. At the end, print:
   - total drafts queued (path to each prospect MD file with a new
     `## Draft (pending approval)` section)
   - any failures

## Output

A summary plus the path to each new draft in `$PROSPECT_DIR/`. In v1,
the Composer writes the draft to the prospect MD file only. The
operator (or, when wired, `tools/sync_drafts.py`) materializes the
draft into Outlook Drafts; the operator then reviews and clicks Send.
See `skills/outlook-ops/SKILL.md` for the bridge options.

$ARGUMENTS
