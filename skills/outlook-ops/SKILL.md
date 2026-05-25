---
name: outlook-ops
description: How agents read from and write to Outlook via Microsoft Graph. Composer uses it to save Drafts. Cadence uses it to detect sends and replies. Read-only patterns work today via the M365 MCP; write paths are stubbed until a Graph write MCP or a small helper is wired.
---

# Outlook Ops

## What works today (read-only)

The session's M365 MCP exposes these read operations:

- `outlook_email_search` — search the user's mailbox by query
  (sender, recipient, subject, body, date range)
- `outlook_calendar_search` — read calendar events
- `find_meeting_availability` — find free/busy slots
- `sharepoint_search` / `sharepoint_folder_search` — surface existing
  Synozur intel

The Cadence agent uses these to:

- Detect a send: search sent-items for the draft's subject and recipient
- Detect a reply: search inbox for replies on the original thread
- Propose meeting slots: find calendar gaps in the next 5 business days

## What does not work today (write — needs wiring)

The current MCP set has no `outlook_send_mail` or `outlook_save_draft`
tool. Until one is wired, the Composer's "save to Outlook Drafts" step
runs via one of three options (v1 supports option C):

**Option A — Graph SDK helper script.** Add a small Python helper at
`tools/outlook_draft.py` that authenticates to Graph (delegated auth,
the user's signed-in identity) and creates a draft via
`POST /me/messages` with a JSON body containing `subject`, `body`, and
`toRecipients`. Messages created via this endpoint default to draft
state (the `isDraft` flag is set true by Graph until the message is
explicitly sent). The Composer invokes the helper via a tool wrapper.
Requires app registration in the Synozur tenant with `Mail.ReadWrite`.

**Option B — A write-side MCP.** Wire a Graph MCP server (e.g. an
internal Synozur MCP or one of the community Graph MCPs) into the
session's MCP config. The Composer then calls
`mcp__graph__save_draft({to, subject, body})`. Cleanest long-term path.

**Option C — Human bridge (v1 default).** The Composer writes the
draft to the prospect MD file under `## Draft (pending approval)`. A
tiny `tools/sync_drafts.py` script (the operator runs it manually or on
a cron) walks the prospect directory, finds any `## Draft (pending
approval)` sections not yet in Outlook, and creates the Draft via the
Graph SDK using the operator's authenticated session. The Composer
itself does not write to Outlook.

For first-generation v1, **use Option C**. It removes the agent's write
authority over Outlook entirely. The agent writes markdown; a sync
script materializes markdown into Drafts.

## The draft body format

When the Composer writes the `## Draft (pending approval)` section, it
must use this exact structure so the sync script can parse it:

```
## Draft (pending approval) — <ISO 8601 timestamp>

**Channel:** email
**To:** <email>
**Subject:** <subject>

<body — plain text, no markdown, no signature; the sync script appends
the signature configured in the operator's Outlook>

---
**Reasoning trace:**
- ...
```

The sync script reads from `**To:**` through to the `---` divider. The
reasoning trace is for humans and stays in the MD file only.

## Detecting a send (Cadence)

```
outlook_email_search:
  folder: sent-items
  to: <recipient email>
  subject: <draft subject>
  after: <draft timestamp>
```

If exactly one match → mark `sent` with `sent_at: <message timestamp>`
and `thread_id: <conversationId from the result>`.
If zero matches and the draft is older than 7 days → set
`hold_reason: draft never sent`, do not transition.
If multiple matches → take the most recent and log a warning.

## Detecting a reply (Cadence)

```
outlook_email_search:
  folder: inbox
  conversationId: <thread_id from the lead>
  from: <prospect email>
  after: <sent_at from the lead>
```

If a match → transition to `state: replied`, copy the reply body into
the lead's `## Conversation` section, classify with the Composer per
`skills/cadence-rules/`.

## Authentication notes

All Graph access is delegated, using the operator's M365 sign-in. The
agent does not hold a service-principal token. This intentionally caps
blast radius: if the harness goes wrong, it can only do what the
signed-in user could already do, and the audit shows it under that
user.
