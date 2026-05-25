---
name: prospect-files
description: The markdown file schema for prospects. One file per lead at $PROSPECT_DIR/<slug>.md. Source of truth for state, dossier, drafts, conversation, and audit log. Read and written by all three agents.
---

# Prospect Files

One markdown file per lead. The file is the prospect.

## Location

All prospect files live at `$PROSPECT_DIR/` (set in `.env`). v1 expects
this to be a OneDrive-synced folder so multiple agents on the operator's
machine see the same state without a database.

## File naming

`<firstname>-<lastname>.md`, all lowercase, hyphenated, ASCII only.
Slugs collide on common names → append a short company suffix:
`jane-smith-acme.md`.

## The schema

```markdown
---
# Identity
name: Jane Smith
email: jane@acme.com
company: Acme
title: VP of Revenue Operations
linkedin_url: https://www.linkedin.com/in/janesmithacme
domain: acme.com
timezone: America/New_York

# State machine
state: new
state_updated_at: 2026-05-25T14:30:00Z
last_agent: null

# Campaign membership
campaign: outbound-email-v1
lawful_basis: legitimate-interest  # or: consent, contract

# Cadence runtime
touch_count: 0
next_step_at: null
thread_id: null
sent_at: null
cadence_template_id: null

# Compliance flags
do_not_contact: false
email_invalid: false

# Outcomes (filled in over time)
icp_score: null
disqualify_reason: null
hold_reason: null
re_engage_at: null
---

# Jane Smith — VP RevOps, Acme

## Dossier
(filled by Prospector)

## Signals
(filled by Prospector)

## ICP Fit
(filled by Prospector)

## Hook
(filled by Prospector)

## Prior Touches
(filled by Prospector)

## Notes
(filled by any agent)

## Draft (pending approval)
(filled by Composer when state transitions to draft_pending_approval;
removed or moved to ## Conversation when the lead transitions to sent)

## Conversation
(filled by Cadence as sends and replies happen; chronological)

## Audit
(append-only log of every state transition and tool call)
```

## State transition rules

Each transition writes:

- `state:` → new value
- `state_updated_at:` → ISO 8601 timestamp
- `last_agent:` → the agent that made the transition
- An entry in `## Audit` like:
  `2026-05-25T14:32:01Z [prospector] state: new → researched (icp_score: 8)`

Agents may only write the transitions named in their definition:

- Prospector: `new → researched | disqualified`
- Composer: `researched | replied → draft_pending_approval`
- Cadence: everything else

## Reading the file

When an agent reads a file:

1. Parse the YAML frontmatter.
2. Check `do_not_contact` first; if true, skip.
3. Check `state` matches the agent's input states; if not, skip.
4. Read the body sections relevant to the agent's job (Prospector reads
   only frontmatter to decide whether to research; Composer reads
   Dossier + Hook + Signals to draft; Cadence reads Conversation + Audit
   to decide next move).

## Writing the file

Always use `Edit` (not `Write`) on existing files to preserve any
operator manual edits. Use `Write` only on the very first creation of
a `state: new` file.

When writing a new section (e.g. Composer adding a `## Draft (pending
approval)`), insert it in the canonical order shown in the schema.
Sections never duplicate — if `## Draft (pending approval)` already
exists, update it in place rather than adding a second one.

## PII and version control

Prospect files contain PII (names, emails, dossier intel). The repo
gitignores `prospects/*.md` except for the schema, template, and
example. Real prospect data never enters the repo — it lives only in
the synced OneDrive folder.
