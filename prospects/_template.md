---
# Identity — fill these in
name: TODO
email: TODO@example.com
company: TODO
title: TODO
linkedin_url: TODO
domain: example.com
timezone: America/New_York

# State machine — leave at `new` for a fresh lead
state: new
state_updated_at: TODO  # ISO 8601, set on creation
last_agent: operator

# Campaign — pick from skills/cadence-rules/templates/*.md
campaign: outbound-email-v1
lawful_basis: legitimate-interest

# Cadence runtime — agents fill these in
touch_count: 0
next_step_at: null
thread_id: null
sent_at: null
cadence_template_id: null

# Compliance flags
do_not_contact: false
email_invalid: false

# Outcomes — populated by agents over time
icp_score: null
disqualify_reason: null
hold_reason: null
re_engage_at: null
---

# TODO Name — TODO Title, TODO Company

<!--
Drop this file into $PROSPECT_DIR and run /run-prospector. The
Prospector will fill in Dossier, Signals, ICP Fit, Hook, and Prior
Touches, and transition the state to `researched` or `disqualified`.

Sections below are intentionally empty — the agents add them as the lead
progresses.
-->
