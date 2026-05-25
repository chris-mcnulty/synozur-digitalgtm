---
name: cadence-rules
description: When to follow up, what to send, and when to stop. Defines pre-approved templated cadence steps that the Cadence agent can auto-fire under guardrails. Used by the Cadence agent.
---

# Cadence Rules

The Cadence agent uses this skill to decide:

1. When to fire the next step (timing in `timing.md`)
2. What content to send (templates in `templates/`)
3. When to stop bothering a prospect (`stop-conditions.md`)

## Files in this skill

- `timing.md` — business-hour windows, timezone rules, business-day math
- `templates/` — pre-approved cadence step templates, each in its own
  file with a stable ID. Changes to a template are PRs.
- `stop-conditions.md` — the rules that move a lead to `dormant`

## The contract with the Composer

The Composer authors first-touch and reply drafts that require human
approval. The Cadence agent only sends content from
`skills/cadence-rules/templates/`. That's the "pre-approved at design
time" promise: each template was reviewed and merged via PR; the auto-send
is acceptable because the wording already passed review.

If a template needs a tweak for a specific prospect, the Cadence agent
**does not tweak**. It saves the rendered version to Outlook Drafts with
`state: draft_pending_approval` and a `hold_reason: needs personalization`
so the Composer (or a human) can handle it.

## Template format

Each file in `templates/` describes **one complete cadence** (a series
of steps under a single campaign), not a single step. The frontmatter
identifies the cadence and enumerates its steps:

```yaml
---
id: outbound-email-v1            # stable cadence ID — referenced by
                                  # prospect frontmatter as
                                  # cadence_template_id
campaign: outbound-email-v1      # campaign tag (usually equals id)
description: Default outbound cadence for cold prospects.
steps:
  - id: outbound-email-v1-step-2 # stable per-step ID
    days_after_previous: 3
    channel: email
  - id: outbound-email-v1-step-3
    days_after_previous: 4
    channel: email
  - id: outbound-email-v1-step-4
    days_after_previous: 7
    channel: email
variables:                       # variables any step in this cadence
  - first_name                   # may reference
  - hook_summary
---
```

Below the frontmatter, the body has one `## Step N` section per step
in `steps`, containing the templated copy for that step. Variables are
referenced as `{first_name}` etc. The Cadence agent fills variables
from the prospect's MD file frontmatter and dossier sections.

When a prospect's frontmatter says `cadence_template_id: outbound-email-v1`,
the Cadence agent looks up `templates/outbound-email-v1.md`, finds the
next step (per `steps[]` and the lead's `touch_count`), and renders
the corresponding `## Step N` body section.

## Adding a new template

1. Create the file under `templates/`.
2. Open a PR. The voice owner reviews; another rep reviews.
3. After merge, the template ID is available for new campaigns. Existing
   in-flight leads keep using the template version they started on
   (immutable cadence — changing mid-stream changes the experiment).
