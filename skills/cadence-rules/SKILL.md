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

Each file in `templates/` has YAML frontmatter:

```yaml
---
id: outbound-email-v1-step-2
campaign: outbound-email-v1
step: 2
days_after_previous: 3
channel: email
description: Day-3 nudge that references the original hook.
variables:
  - first_name
  - hook_summary
---
```

Followed by the body in markdown. Variables are referenced as
`{first_name}` etc. The Cadence agent fills variables from the prospect's
MD file frontmatter and dossier sections.

## Adding a new template

1. Create the file under `templates/`.
2. Open a PR. The voice owner reviews; another rep reviews.
3. After merge, the template ID is available for new campaigns. Existing
   in-flight leads keep using the template version they started on
   (immutable cadence — changing mid-stream changes the experiment).
