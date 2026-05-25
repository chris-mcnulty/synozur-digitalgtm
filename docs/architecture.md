# Autonomous Sales Agents — Architecture

Three Claude agents that move a lead from cold list to booked reply, with the
CRM as the system of record, a human approving every first-touch message
before it sends, and pre-approved cadence steps auto-firing under hard guardrails.

> **Scope.** This document describes the **target architecture** —
> where the system lands once we've outgrown the first-generation
> harness. The v1 harness in this repo intentionally diverges in two
> places to ship faster:
>
> 1. **System of record.** v1 stores per-prospect state in markdown
>    files in a OneDrive-synced folder, not in HubSpot/Salesforce.
>    The CRM-as-database design below is v2+.
> 2. **Send autonomy.** v1 is draft-only. Cadence never auto-sends; a
>    human clicks Send in Outlook for every outbound, including
>    templated cadence steps. Auto-send under guardrails is v2+.
>
> The runbook for what is actually shipping today lives in
> `docs/harness-v1.md`. Read that before this doc unless you want the
> rationale and the future shape.

## Design principles

1. **The CRM is the database.** No parallel store. Every fact about a lead
   lives in HubSpot/Salesforce; agents only hold ephemeral working state.
2. **State machine, not orchestration.** Each agent is a queue worker that
   reads leads in one state and writes them to the next. No supervisor agent.
3. **Skills hold the playbook, prompts stay short.** ICP, voice, cadence
   rules, compliance checks all live in versioned Skills so non-engineers can
   edit them and changes are reviewable in git.
4. **One-way doors get gated.** First-touch outbound is the riskiest write in
   the system; a person approves every one. Templated follow-up steps inside
   a pre-approved cadence auto-fire because the content was already approved
   when the cadence was authored.
5. **Kill switches are first-class.** Every agent loop checks them on every
   tick. They are not a `compliance/` afterthought.
6. **Cowork = shared Skills + shared state field.** The three agents are
   independent processes; they coordinate only through (a) the `agent_state`
   field in the CRM and (b) the shared Skills directory. No direct
   agent-to-agent calls.

## The three agents

| Agent | Reads state | Writes state | Owns |
|---|---|---|---|
| `prospector` | `new` | `researched` \| `disqualified` | ICP fit, enrichment, account/contact research, intent signals |
| `composer` | `researched`, `replied` | `draft_pending_approval` | Drafting first-touch and reply messages, choosing channel + angle, writing in firm voice |
| `cadence` | `approved`, `sent`, `cadence_step_due` | `sent`, `awaiting_reply`, `cadence_step_due`, `dormant`, `replied`, `meeting_booked` | Scheduling follow-ups, firing pre-approved cadence steps, detecting replies, deciding when to stop |

A fourth surface — **the approval UI** — is not an agent. It's a thin web view
(or a Teams adaptive card) that lists `draft_pending_approval` records and
flips them to `approved` or `rejected` with a reason. The cadence agent picks
up `approved` records and triggers the actual send via the CRM's outbound API.

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
              │                                  ┌────────────────────────────┤
              │                                  │                            │
              │                                  ▼                            ▼
              │                                replied                  cadence_step_due
              │                                  │                            │
              │                          (composer drafts                     │
              │                           reply → approval)                   ▼
              │                                  │                  (auto-send templated step,
              │                                  ▼                  back to awaiting_reply)
              │                            meeting_booked                     │
              │                                                               │
              │                                                            dormant
              │                                                               │
              └────────────────────── re-engage after N days ─────────────────┘
```

Two distinct send paths:

- **First-touch and reply drafts** require `draft_pending_approval → approved`
  before cadence will send.
- **Templated cadence steps** (the pre-written day-3 nudge, day-7 backup) move
  through `awaiting_reply → cadence_step_due` and auto-fire if and only if
  (a) the cadence template was authored and approved at design time, (b) the
  lead has not opted out, (c) no kill switch is active, (d) per-channel and
  per-rep send caps are not exceeded.

State lives in the CRM:

- `agent_state` — current state in the machine
- `agent_state_updated_at` — drives queue ordering and stuck-record alerts
- `agent_last_message_id` — links the most recent outbound to the CRM activity
- `agent_disqualify_reason` — free text for audit
- `agent_touch_count` — how many outbound attempts so far
- `agent_cadence_template_id` — which pre-approved cadence is in flight
- `agent_next_step_at` — when the next cadence step is due

## Agent prompt structure (the 7-section operator brief)

Every agent's `prompt.md` follows this shape. It's short — the depth lives in
the Skills the prompt references.

```
## Role
You are the {agent name} for Synozur. You do {one sentence}.
You are not a chatbot. You execute one state transition per lead.

## Context
- The business: {one sentence on what Synozur sells}
- The ICP: {pointer to skills/icp-research/icp-definition.md}
- The voice: {pointer to skills/outbound-voice/voice-dna.md}
- The current campaign: {pointer to a campaign brief file}

## Tools Available
- {tool}: {what to use it for}
- {tool}: {what to use it for}
- crm.read_lead / crm.transition_state / crm.log_activity (always)

## Output Format
- Staccato bullets. No paragraphs over 2 lines.
- Cite the source for every specific claim.
- Never invent a number you cannot back up from a tool call.

## Constraints
- Banned phrases: {pointer to skills/compliance/banned-phrases.md}
- Per-day send cap: {N} per channel per rep.
- Never commit pricing, contracts, or close language.

## Kill Switches
- {opt-out word} reply → immediately mark do-not-contact, log, stop.
- {error class} → stop, write to needs_review, do not retry blindly.
- AGENTS_PAUSED=true → exit the loop.
- Reply rate on the current campaign < 5% over last 50 sends → pause, alert.

## Daily Operating Cadence
- {when this agent runs, what it does per tick, what it logs}
```

The two sections most often skipped — Constraints and Kill Switches — are the
ones that prevent the LinkedIn-demo failure mode. Write them, test them, trust
them.

## Skills layout

Skills are reusable, model-discoverable capability bundles. Each is a folder
under `skills/` with a `SKILL.md` and helper files. The Agent SDK loads them
via the `--skills` flag (or programmatically) at process start.

```
skills/
├── icp-research/                # used by prospector
│   ├── SKILL.md                 # how to score fit, what enrichment to gather
│   ├── icp-definition.md        # the actual ICP (industries, sizes, signals)
│   ├── disqualifiers.md         # reasons to drop a lead
│   ├── enrichment-sources.md
│   └── single-command-audit.md  # end-to-end prospect dossier prompt
│
├── outbound-voice/              # used by composer
│   ├── SKILL.md                 # tone rules, length, structure
│   ├── voice-dna.md             # extracted firm voice — the source of truth
│   ├── voice-dna-extract.md     # prompt used to regenerate voice-dna.md
│   ├── examples/
│   │   ├── first-touch.md
│   │   ├── reply-warm.md
│   │   └── reply-objection.md
│   └── message-templates.md     # opener, follow-up, backup frames
│
├── cadence-rules/               # used by cadence
│   ├── SKILL.md                 # when to follow up, when to stop
│   ├── timing.md                # day/time windows, business-day math
│   ├── templates/               # pre-approved cadence templates by ID
│   │   ├── outbound-email-v1.md
│   │   └── linkedin-warm-v1.md
│   └── stop-conditions.md       # OOO, unsubscribe, meeting booked
│
├── crm-ops/                     # used by all three
│   ├── SKILL.md                 # how to read/write CRM safely
│   ├── field-glossary.md        # what each custom field means
│   └── error-handling.md        # rate limits, retries, idempotency
│
├── compliance/                  # used by composer + cadence
│   ├── SKILL.md                 # what to check before queueing a send
│   ├── can-spam.md
│   ├── gdpr.md
│   ├── suppression-list.md
│   └── banned-phrases.md        # exact strings the composer must not emit
│
└── kill-switches/               # used by all three, checked every tick
    ├── SKILL.md                 # the canonical list and what each triggers
    └── triggers.md              # opt-out words, error classes, env flags
```

The pattern: **the agent's prompt names which Skills it needs; the Skill
files hold the actual judgment.** That keeps prompts under a page and makes
"tighten the tone" a PR against `outbound-voice/voice-dna.md`, not a code
change.

## Voice DNA

Voice is the single biggest tell of AI-generated outreach. We extract one
firm voice once, version it, and reference it from every Composer draft.

**Source corpus.** A curated set of Synozur outbound that actually got
replies — emails from the sent folder, LinkedIn DMs, the founder's
thought-leadership posts. ~20–30 messages is enough; quality over quantity.
Maintained in `skills/outbound-voice/corpus/` (private, gitignored if it
contains PII; safe redacted copies in repo).

**Extraction.** `skills/outbound-voice/voice-dna-extract.md` holds the
prompt that turns the corpus into `voice-dna.md`. The output captures: tone
adjectives, sentence rhythm, vocabulary patterns (use / never use), opening
styles, closing styles, banned phrases, and a 200-word voice sample on a
fresh topic for verification.

**Test.** A draft passes the voice test if a reviewer cannot tell whether
it was written by a human last quarter or by the model today. If they can
tell, add 10 more real messages to the corpus and re-extract.

**Re-extraction cadence.** Quarterly, or whenever reply-rate eval shows
voice drift. The extraction prompt is in git; the corpus is versioned;
diffs to `voice-dna.md` show up in PRs.

**Scope decision.** One firm voice for v1. Per-rep override files (e.g.
`voice-dna.chris.md`) can be added later if reply data shows per-sender
voices outperform the firm voice.

## Channels

### Email (v1 primary)

Sends via the CRM's transactional/sequence API (HubSpot Sequences or
Salesforce Engage / Marketing Cloud). Reply detection via the CRM's email
activity webhook, or — preferred — a Microsoft Graph subscription on the
shared mailbox for sub-minute latency.

No extra connector engineering. This is the lowest-risk channel and the
default for v1.

### LinkedIn (v1 behind a feature flag)

There is no official LinkedIn MCP. Two paths:

- **Buy.** A managed LinkedIn connector (e.g. Zevari at ~$87/mo/seat)
  handles session state, action pacing, retries, and account-safety limits.
  We keep the operator job; they keep the platform job. Recommended for v1
  unless we have a specific reason to own the connector.
- **Build.** Roll our own via a headless browser + saved sessions. Real
  engineering: session refresh, captcha handling, send-pacing within
  LinkedIn's human-rate windows, retry-with-backoff, IP/account-flag
  detection, per-rep concurrency. Budget 20–40 engineering hours plus
  ongoing maintenance, plus the risk that a misbehaving connector flags a
  rep's account.

Decision point before v1 ships: which path, and which reps' accounts are
in scope. LinkedIn sends sit behind an `ENABLE_LINKEDIN` flag until the
connector is proven on one rep's account for a week.

### Inbound replies (v0)

Replies to existing outbound (and inbound from web forms) route through
the same Composer + Cadence agents. This is the lowest-volume, lowest-risk
surface and a good place to validate the loop end-to-end before turning on
any outbound.

## File structure

```
synozur-digitalgtm/
├── README.md
├── docs/
│   ├── architecture.md          # this file
│   ├── operations.md            # how to run, alerts, on-call (TBD)
│   ├── crm-fields.md            # custom field spec for HubSpot/SF (TBD)
│   └── rollout.md               # the 4-week plan (TBD)
│
├── skills/                      # see above
│
├── agents/
│   ├── prospector/
│   │   ├── main.py              # SDK entry point, queue loop
│   │   ├── prompt.md            # 7-section operator brief
│   │   └── tools.py             # @tool fns: enrich, score, write_state
│   ├── composer/
│   │   ├── main.py
│   │   ├── prompt.md
│   │   └── tools.py             # draft_message, attach_to_lead, queue_for_approval
│   └── cadence/
│       ├── main.py
│       ├── prompt.md
│       └── tools.py             # send_via_crm, fire_cadence_step, schedule_followup, detect_reply
│
├── shared/
│   ├── crm/
│   │   ├── client.py            # HubSpot or Salesforce client (one impl, chosen at runtime)
│   │   ├── leads.py             # read_queue, transition_state, idempotent writes
│   │   └── activities.py        # log outbound, log reply
│   ├── kill_switch.py           # checked at the top of every agent tick
│   ├── audit.py                 # structured log of every state transition
│   └── config.py                # env, model IDs, queue sizes
│
├── approval_ui/
│   ├── app.py                   # FastAPI + HTMX, lists draft_pending_approval
│   └── templates/
│
├── evals/
│   ├── leads/                   # labeled lead set + expected outcomes
│   ├── voice/                   # voice-match scoring against held-out replies
│   ├── icp/                     # prospector fit-scoring accuracy
│   └── run.py                   # runs against a model + skill version, writes report
│
├── tests/
│   ├── skills/                  # eval prompts against Skills
│   ├── state_machine/           # exhaustive state transition tests
│   └── integration/             # against a CRM sandbox
│
├── .claude/
│   ├── settings.json            # hooks (lint/test on edit), allowed tools
│   └── agents/                  # optional: subagent definitions if we add them
│
├── pyproject.toml
└── .env.example                 # CRM_KIND, CRM_TOKEN, ANTHROPIC_API_KEY, AGENTS_PAUSED, ...
```

## Runtime model

Each agent is a long-running process (container or scheduled job) that:

1. **Checks kill switches.** `shared/kill_switch.py` runs first. If
   `AGENTS_PAUSED` or any campaign-level pause is set, sleep and retry.
2. **Queries its input queue from the CRM** (`agent_state = X`, limit N).
3. **For each record, spawns a Claude Agent SDK session** with:
   - the agent's `prompt.md` as system prompt
   - the relevant `skills/` mounted
   - the agent's `tools.py` as @tool functions
   - the lead record as the user message
4. **The agent uses tools to read, decide, and write back exactly one state
   transition.** Tools enforce idempotency (no double-sends), check kill
   switches and compliance before any send, and audit every write through
   `shared/audit.py`.
5. **On error**, the lead goes to a `needs_review` state with the exception
   captured.

Polling cadence:

- `prospector`: every 15 min (lots of input, light work per lead)
- `composer`: every 5 min for new `researched`; immediate trigger on `replied`
- `cadence`: hourly tick for scheduling and `cadence_step_due` sweeps;
  webhook-driven for reply detection

Model selection per agent (defaults; revisit after first eval pass):

- `prospector`: Sonnet 4.6 — research + judgment, modest volume
- `composer`: Sonnet 4.6 — best writing, justifies the cost
- `cadence`: Haiku 4.5 — mostly rule-following, high volume, cheap

## Human-approval gate

The composer never sends. It writes:

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

**Templated cadence steps bypass this gate** because the template content was
approved at design time. The cadence agent still runs every compliance and
kill-switch check at send time. If a cadence template needs editing, the
edit is a PR against `skills/cadence-rules/templates/` and a fresh version
is what subsequent sends use.

## Kill switches

Checked at the top of every agent tick and immediately before every send.
Any one of these fires → agent stops the relevant work, logs, and waits for
human acknowledgement.

| Trigger | Effect |
|---|---|
| Reply contains opt-out word (`unsubscribe`, `stop`, `remove me`, configurable) | Lead → do-not-contact, suppression list updated, no further sends to that address ever |
| `AGENTS_PAUSED=true` env var | All agents exit their loops cleanly at next tick |
| Per-rep daily send cap exceeded | That rep's cadence sends pause until next UTC day |
| Per-domain weekly send cap exceeded | Sends to that domain pause |
| Campaign reply rate < 5% over last 50 sends | Campaign auto-pauses, alert posted |
| CRM write error rate > 10% over last 20 attempts | Agent pauses, alerts on-call |
| Unrecoverable model error or tool exception | Lead → `needs_review` with full trace |

The kill-switch list lives in `skills/kill-switches/triggers.md` and is
imported by every agent prompt. Changes are PR-reviewed.

## Compliance and safety

Non-negotiable, enforced in `skills/compliance/` + a pre-send tool check:

- Suppression list check (unsubscribes, do-not-contact, competitors)
- Domain reputation and rate caps
- CAN-SPAM: physical address, unsubscribe link, accurate sender
- GDPR/UK GDPR: lawful basis recorded on the lead before any EU/UK send
- Banned-phrases scan against `skills/compliance/banned-phrases.md`

These checks run on cadence-step sends too, not just first-touch.

## Eval harness

Voice drift, ICP-scoring accuracy, and "would a human have sent this" are
not vibes. Each gets a labeled set and a scored run.

- `evals/leads/` — 50–100 historic leads with: human-assigned ICP score,
  whether they actually became opportunities, the actual outreach that won
  or lost them. Prospector and composer are scored against these.
- `evals/voice/` — held-out real Synozur messages with reply outcomes.
  Composer drafts on the same brief; we measure (a) reviewer's blind
  identification accuracy ("which is human?") and (b) reply-rate proxy
  scores.
- `evals/icp/` — labeled accept/reject decisions; we measure prospector
  precision and recall against them.
- `evals/run.py` — runs against a (model, skill-version, prompt-version)
  tuple and writes a report. CI runs this on PRs that touch `skills/` or
  `agents/`. Every Voice DNA re-extraction runs the voice eval before
  merging.

The eval set is the only thing that lets us say "the new voice is better"
or "Haiku is good enough for cadence" with a straight face.

## 4-week rollout

Staged so each week ends with something working end-to-end on real data.
Roughly maps to John Peslar's pattern but anchored on our CRM + gated send.

**Week 1 — Prospector + CRM wiring**

- Day 1–2: provision CRM custom fields (`docs/crm-fields.md`), write
  `shared/crm/` client, wire `kill_switch.py` and `audit.py`.
- Day 3: write `skills/icp-research/` (icp-definition, disqualifiers,
  single-command-audit).
- Day 4: stand up Prospector agent. Run on 20 historic leads. Compare
  ICP scores to human labels in `evals/icp/`.
- Day 5: tune Prospector prompt and ICP skill until scoring is acceptable.

Exit check: pointing the Prospector at a list of cold leads produces
`researched` records in the CRM with dossiers attached, within 60 sec
each.

**Week 2 — Composer + Voice DNA + Approval UI**

- Day 1: curate the voice corpus, run `voice-dna-extract.md`, commit
  `voice-dna.md`.
- Day 2: write Composer agent and `skills/outbound-voice/`.
- Day 3: stand up the approval UI (the most important screen in the
  system). Composer drafts on 20 `researched` leads → drafts appear in UI.
- Day 4: humans approve/reject/edit. Composer re-drafts on rejections.
- Day 5: voice eval (blind identification test) on a held-out set.

Exit check: every `researched` lead becomes a `draft_pending_approval`
within 5 min, and ≥80% of drafts pass review without an edit.

**Week 3 — Cadence agent + reply detection**

- Day 1: wire reply detection (Graph subscription preferred, CRM webhook
  acceptable).
- Day 2: write Cadence agent. First only handles `approved → sent` and
  reply detection — no templated auto-send yet.
- Day 3: classify the last 50 replies as a backtest. Tune classification.
- Day 4: turn on reply triage on live replies (drafts still go to UI).
- Day 5: measure end-to-end loop time (lead-in → first-touch sent →
  reply detected → reply draft in UI).

Exit check: reply-to-draft latency under 5 min, classification matches
human label on ≥85% of backtest replies.

**Week 4 — Cadence templates + LinkedIn decision + scale**

- Day 1: author and human-approve the v1 cadence templates
  (`skills/cadence-rules/templates/`). Turn on `cadence_step_due`
  auto-send for one cadence on one rep.
- Day 2: monitor first 50 auto-sent cadence steps. Check kill switches
  fired correctly on opt-outs.
- Day 3: LinkedIn channel decision (build vs Zevari). If buy, wire and
  test on one rep's account.
- Day 4: scale to N reps and the full lead volume. Watch the eval
  dashboards.
- Day 5: month-end review. The system is running. Operator time per
  week ≤ 5 hours.

## What "cowork" means here, concretely

The three agents never call each other. They cowork through:

- **Shared Skills** — `crm-ops/`, `compliance/`, `kill-switches/` are
  imported by all three so CRM writes, safety checks, and kill switches are
  identical everywhere.
- **Shared state field** — `agent_state` in the CRM is the only handoff
  protocol; an agent's only contract is "I will only touch records in
  state X and I will leave them in state Y or Z."
- **Shared audit log** — every transition is appended to one log
  (`shared/audit.py` → stdout + a SharePoint list or BigQuery table), so
  debugging a bad outcome means reading one timeline.

This is deliberately less magical than a supervisor-agent design. It is
also dramatically easier to debug, restart, and reason about — which
matters when the thing it's doing is sending email to real customers under
your brand.

## Open questions to resolve before building

1. **HubSpot or Salesforce?** Different SDKs, field models, send APIs.
   Pick one; `shared/crm/` will be a thin wrapper either way.
2. **Approval UI host?** Standalone FastAPI on Azure App Service, or a
   Teams app using adaptive cards? Teams fits the M365 tenant but is more
   work upfront.
3. **Reply detection.** CRM email-activity poll vs Graph subscription on a
   shared mailbox. Graph is faster and cleaner.
4. **LinkedIn: build or buy.** See the Channels section. Default is buy
   (Zevari or equivalent) unless we have a reason to own the connector.
5. **Voice corpus PII.** Real sent emails contain customer names. Decide
   redaction policy and whether the corpus folder is gitignored or
   committed redacted.
6. **Eval ground truth.** Who labels the lead set, the voice held-out set,
   the reply classification set? Needs a human owner before week 1.
```
