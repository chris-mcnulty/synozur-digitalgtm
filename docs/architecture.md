# Autonomous Sales Agents — Architecture

Three Claude agents that move a lead from cold list to booked reply, with the
CRM as the system of record and a human approving every outbound message before
it sends.

## Design principles

1. **The CRM is the database.** No parallel store. Every fact about a lead
   lives in HubSpot/Salesforce; agents only hold ephemeral working state.
2. **State machine, not orchestration.** Each agent is a queue worker that
   reads leads in one state and writes them to the next. No supervisor agent.
3. **Skills hold the playbook, prompts stay short.** Tone, ICP, cadence rules,
   compliance checks all live in versioned Skills so non-engineers can edit
   them and so changes are reviewable in git.
4. **One-way door = human gate.** Sending an email or LinkedIn message is the
   only irreversible step. A person approves it. Everything else is autonomous.
5. **Cowork = shared Skills + shared store.** The three agents are independent
   processes; they coordinate only through (a) the CRM state field and (b) the
   shared Skills directory. No direct agent-to-agent calls.

## The three agents

| Agent | Reads state | Writes state | Owns |
|---|---|---|---|
| `prospector` | `new` | `researched` \| `disqualified` | ICP fit, enrichment, account/contact research, intent signals |
| `composer` | `researched`, `reply_received` | `draft_pending_approval` | Drafting first-touch and reply messages, choosing channel + angle |
| `cadence` | `approved`, `sent`, `awaiting_reply` | `sent`, `awaiting_reply`, `dormant`, `replied`, `meeting_booked` | Scheduling follow-ups, detecting replies, deciding when to stop |

A fourth surface — **the approval UI** — is not an agent. It's a thin web view
(or a Teams adaptive card) that lists `draft_pending_approval` records and
flips them to `approved` or `rejected` with a reason. Cadence picks up
`approved` records and triggers the actual send via the CRM's outbound API.

## State machine

```
                          ┌──────────────┐
              ┌──────────▶│ disqualified │
              │           └──────────────┘
              │
   new ──▶ researched ──▶ draft_pending_approval ──▶ approved ──▶ sent ──▶ awaiting_reply
              ▲                       │                                       │
              │                       └──▶ rejected ──▶ (back to researched)  │
              │                                                               │
              │                       ┌───────────────────────────────────────┤
              │                       │                                       │
              │                       ▼                                       ▼
              │                   replied ──▶ (composer drafts reply)      dormant
              │                       │                                       │
              │                       ▼                                       │
              │                  meeting_booked                               │
              │                                                               │
              └───────────────────── re-engage after N days ──────────────────┘
```

State lives in a single CRM custom field (`agent_state`) plus a few
companions:

- `agent_state_updated_at` — drives queue ordering and stuck-record alerts
- `agent_last_message_id` — links the most recent outbound to the CRM activity
- `agent_disqualify_reason` — free text for audit
- `agent_touch_count` — how many outbound attempts so far

Picking each agent's queue is then just a CRM list/SOQL query like
`agent_state = 'researched' AND owner = <agent user>`.

## Skills layout

Skills are reusable, model-discoverable capability bundles. Each one is a
folder under `skills/` with a `SKILL.md` and optional helper files. The Agent
SDK loads them via the `--skills` flag (or programmatically) at process start.

```
skills/
├── icp-research/          # used by prospector
│   ├── SKILL.md           # how to score fit, what enrichment to gather
│   ├── icp-definition.md  # the actual ICP (industries, sizes, signals)
│   ├── disqualifiers.md   # reasons to drop a lead
│   └── enrichment-sources.md
│
├── outbound-voice/        # used by composer
│   ├── SKILL.md           # tone rules, length, structure
│   ├── examples/
│   │   ├── first-touch.md
│   │   ├── reply-warm.md
│   │   └── reply-objection.md
│   └── forbidden-phrases.md
│
├── cadence-rules/         # used by cadence
│   ├── SKILL.md           # when to follow up, when to stop
│   ├── timing.md          # day/time windows, business-day math
│   └── stop-conditions.md # OOO, unsubscribe, meeting booked
│
├── crm-ops/               # used by all three
│   ├── SKILL.md           # how to read/write CRM safely
│   ├── field-glossary.md  # what each custom field means
│   └── error-handling.md  # rate limits, retries, idempotency
│
└── compliance/            # used by composer + cadence
    ├── SKILL.md           # what to check before queueing a send
    ├── can-spam.md
    ├── gdpr.md
    └── suppression-list.md
```

The pattern: **the agent prompt names which Skills it needs; the Skill files
hold the actual judgment.** That keeps prompts under a page and makes
"tighten the tone" a PR against `outbound-voice/`, not a code change.

## File structure

```
synozur-digitalgtm/
├── README.md
├── docs/
│   ├── architecture.md          # this file
│   ├── operations.md            # how to run, alerts, on-call (TBD)
│   └── crm-fields.md            # custom field spec for HubSpot/SF (TBD)
│
├── skills/                      # see above
│
├── agents/
│   ├── prospector/
│   │   ├── main.py              # SDK entry point, queue loop
│   │   ├── prompt.md            # system prompt, references Skills by name
│   │   └── tools.py             # @tool fns: enrich, score, write_state
│   ├── composer/
│   │   ├── main.py
│   │   ├── prompt.md
│   │   └── tools.py             # draft_message, attach_to_lead, queue_for_approval
│   └── cadence/
│       ├── main.py
│       ├── prompt.md
│       └── tools.py             # send_via_crm, schedule_followup, detect_reply
│
├── shared/
│   ├── crm/
│   │   ├── client.py            # HubSpot or Salesforce client (one impl, chosen at runtime)
│   │   ├── leads.py             # read_queue, transition_state, idempotent writes
│   │   └── activities.py        # log outbound, log reply
│   ├── audit.py                 # structured log of every state transition
│   └── config.py                # env, model IDs, queue sizes
│
├── approval_ui/
│   ├── app.py                   # FastAPI + HTMX, lists draft_pending_approval
│   └── templates/
│
├── tests/
│   ├── skills/                  # eval prompts against Skills (does composer respect tone?)
│   ├── state_machine/           # exhaustive state transition tests
│   └── integration/             # against a CRM sandbox
│
├── .claude/
│   ├── settings.json            # hooks (lint/test on edit), allowed tools
│   └── agents/                  # optional: subagent definitions if we add them
│
├── pyproject.toml
└── .env.example                 # CRM_KIND, CRM_TOKEN, ANTHROPIC_API_KEY, ...
```

## Runtime model

Each agent is a long-running process (container or scheduled job) that:

1. Queries its input queue from the CRM (`agent_state = X`, limit N).
2. For each record, spawns a Claude Agent SDK session with:
   - the agent's `prompt.md` as system prompt
   - the relevant `skills/` mounted
   - the agent's `tools.py` as @tool functions
   - the lead record as the user message
3. The agent uses tools to read, decide, and write back exactly one state
   transition. Tools enforce idempotency (no double-sends) and audit every
   write through `shared/audit.py`.
4. On completion, the loop moves to the next record. On error, the lead goes
   to a `needs_review` state with the exception captured.

Polling cadence:

- `prospector`: every 15 min (lots of input, light work per lead)
- `composer`: every 5 min for new `researched`; immediate trigger on `replied`
- `cadence`: hourly tick for follow-up scheduling; webhook-driven for reply
  detection if the CRM supports it

## Human-approval gate (the only one-way door)

The `composer` never sends. It writes:

- the draft to a CRM note attached to the lead, AND
- a record to `draft_pending_approval` state

The approval UI:

- Lists drafts oldest-first, grouped by sender
- Shows: lead context, reasoning trace (from the audit log), the draft, a
  diff against the last version if it was rejected before
- Buttons: **Approve**, **Edit & approve**, **Reject with reason**
- On approve → state becomes `approved`; cadence picks it up within minutes
  and calls the CRM's send API
- On reject → state goes back to `researched` with the rejection reason
  appended to the lead; composer re-drafts with that feedback in context

This is the single most important piece. Build it before the agents are
"smart" — a dumb composer with a working approval loop is more useful than a
clever composer without one.

## Compliance and safety

Non-negotiable, enforced in `compliance/` skill + a pre-send tool check:

- Suppression list check (unsubscribes, do-not-contact, competitors)
- Domain reputation and rate caps (max N sends per domain per week)
- CAN-SPAM: physical address, unsubscribe link, accurate sender
- GDPR/UK GDPR: lawful basis recorded on the lead before any EU/UK send
- Per-rep daily send cap (so one bug can't fire 10,000 emails)
- Kill switch: a single env var (`AGENTS_PAUSED=true`) that every agent
  checks at the top of its loop

## What "cowork" means here, concretely

The three agents never call each other. They cowork through:

- **Shared Skills** — `crm-ops/`, `compliance/` are imported by all three so
  CRM writes and safety checks are identical everywhere
- **Shared state field** — `agent_state` in the CRM is the only handoff
  protocol; an agent's only contract is "I will only touch records in state
  X and I will leave them in state Y or Z"
- **Shared audit log** — every transition is appended to one log (`shared/
  audit.py` → stdout + a SharePoint list or BigQuery table), so debugging a
  bad outcome means reading one timeline

This is deliberately less magical than a supervisor-agent design. It is also
dramatically easier to debug, restart, and reason about — which matters when
the thing it's doing is sending email to real customers under your brand.

## Open questions to resolve before building

1. **HubSpot or Salesforce?** Different SDKs, different field models, different
   send APIs. Pick one; `shared/crm/` will be a thin wrapper either way.
2. **Approval UI host?** Standalone FastAPI on Azure App Service, or a Teams
   app using adaptive cards? The Teams route fits the M365 tenant but is more
   work upfront.
3. **Reply detection.** Polling the CRM's email activity vs. a Graph
   subscription on the shared mailbox. Graph is faster and cleaner.
4. **Model choice per agent.** Default to Sonnet 4.6 for composer (best
   writing), Haiku 4.5 for cadence (cheap, mostly rule-following), Sonnet 4.6
   for prospector. Revisit after first eval pass.
5. **Eval harness.** What does "good" look like for each agent? Need a small
   labeled set of leads + expected outcomes before we tune prompts or Skills.
