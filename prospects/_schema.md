# Prospect File Schema

This file documents the prospect MD file format. It is committed to the
repo as reference. Real prospect files live in `$PROSPECT_DIR/` (a
OneDrive-synced folder) and are gitignored.

For the authoritative description of how each agent reads and writes
these files, see `skills/prospect-files/SKILL.md`.

For the working blank template you copy when adding a new prospect, see
`_template.md`.

For a worked example showing the file at every state in the machine,
see `_example-jane-smith.md`.

## Frontmatter fields

| Field | Required | Set by | Notes |
|---|---|---|---|
| `name` | yes | operator | Full name |
| `email` | for sends | operator | Validated by Cadence on first send attempt |
| `company` | yes | operator | |
| `title` | yes | operator or Prospector | |
| `linkedin_url` | recommended | operator | If absent, Prospector searches |
| `domain` | yes | operator | The email domain. Used for suppression matching. |
| `timezone` | recommended | operator | Defaults to US/Eastern if absent |
| `state` | yes | all agents | See state machine in `CLAUDE.md` |
| `state_updated_at` | yes | all agents | ISO 8601 |
| `last_agent` | yes | all agents | `prospector` \| `composer` \| `cadence` \| `operator` |
| `campaign` | yes | operator | Tags the lead into a campaign for stats and pacing |
| `lawful_basis` | for EU/UK | operator | `legitimate-interest` \| `consent` \| `contract`. GDPR. |
| `touch_count` | yes | Cadence | Increments on every send |
| `next_step_at` | when in cadence | Cadence | When the next templated step is due |
| `thread_id` | after first send | Cadence | Outlook conversation ID for reply detection |
| `sent_at` | after first send | Cadence | Timestamp of first send |
| `cadence_template_id` | after enrolled | Cadence | Locks the lead to a template version. Must equal the `id:` in the frontmatter of one of the files in `skills/cadence-rules/templates/` (also equals the filename stem). |
| `draft_subject` | when drafted | Composer | Subject line of the current `## Draft (pending approval)`. Cadence uses it to match the eventual Outlook send back to the lead via sent-items search. |
| `do_not_contact` | yes | Cadence / operator | Permanent if true. Cleared only by an operator editing the prospect file directly, with the reason logged in `## Audit`. Never cleared by an agent. |
| `email_invalid` | optional | Cadence | Set on hard bounce |
| `icp_score` | after research | Prospector | 1–10 or null |
| `disqualify_reason` | when disqualified | Prospector | One sentence |
| `hold_reason` | when held | Composer/Cadence | Why the draft is paused for human review |
| `re_engage_at` | when dormant | Cadence | When to consider revisiting |

Reply-classification details (category, proposed meeting slots, etc.)
are written by Cadence into the `## Notes` body section, not into
frontmatter. Keeping classification out of the schema avoids schema
drift for what is essentially free-form annotation.

## Body sections (in order)

1. `# <Name> — <Title>, <Company>` — header, one line
2. `## Dossier` — identity facts; Prospector
3. `## Signals` — last-30-day intel; Prospector
4. `## ICP Fit` — score + justification; Prospector
5. `## Hook` — the one specific signal to lead with; Prospector
6. `## Prior Touches` — anything Outlook or SharePoint surfaced; Prospector
7. `## Notes` — anything unusual; any agent
8. `## Draft (pending approval)` — the most recent draft; Composer
9. `## Conversation` — chronological send/reply log; Cadence
10. `## Audit` — append-only transition + tool log; all agents

Sections may be absent if the lead hasn't reached that state yet. The
canonical order is the order sections appear in once present.
