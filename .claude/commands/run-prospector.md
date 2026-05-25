---
description: Research every prospect MD file in state=new. Spawns the Prospector subagent on each.
---

# /run-prospector

Run the Prospector against every prospect file in `$PROSPECT_DIR/` whose
YAML frontmatter has `state: new`.

## Steps

1. Read `$PROSPECT_DIR/` (resolve from `.env`). List every `.md` file.
2. For each file, read the YAML frontmatter. Skip unless `state: new`.
3. Skip any file with `do_not_contact: true`.
4. Check kill switches in `skills/kill-switches/triggers.md`. If
   `AGENTS_PAUSED=true`, stop and report.
5. For each remaining file, spawn the `prospector` subagent with the
   file path as the input. Run them sequentially (not in parallel) for
   the first generation — easier to debug and respects search-rate limits.
6. After each run, log: prospect slug, new state, ICP score (if any),
   one-line outcome.
7. At the end, print a summary table:
   - total candidates
   - moved to `researched`
   - moved to `disqualified`
   - errored (moved to `needs_review`)

## Output

A markdown summary to the operator. Do not modify any file the
Prospector did not already write.

$ARGUMENTS
