# Harness v1 — Runbook

This is the operator-facing runbook for the first-generation sales
prospecting harness. Read once. Keep open while running.

## What is built

A repo of agent definitions, slash commands, skills, and a prospect
file schema that together implement the three-agent pattern described
in `docs/architecture.md`, scoped to a first-generation harness:

- One operator on one machine
- Per-prospect markdown files in a OneDrive-synced folder
- Outlook as the send channel — drafts only, human clicks Send
- Read-only Microsoft Graph access today; one small Python sync script
  needed before drafts can be materialized into Outlook automatically

## What is not built (deliberately)

- No production CRM integration. The system of record is the folder of
  markdown files. We chose this for v1 to ship fast; the architecture
  doc (`docs/architecture.md`) describes the eventual HubSpot/Salesforce
  shape.
- No auto-send. Every outbound message is a Draft until a human opens
  Outlook and clicks Send.
- No approval UI beyond Outlook itself. v1's approval surface IS the
  Drafts folder.
- No eval harness yet. `evals/` is a TODO; first-generation operation
  is "watch the output, iterate the skills."
- No Voice DNA extracted yet. The first thing you do before running the
  Composer is curate a corpus and extract.

## Day-zero setup

In order, once:

1. **Clone the repo.** `cd` into it.
2. **Create `.env`.** Copy `.env.example` to `.env`. Set:
   - `PROSPECT_DIR` to your synced OneDrive folder
     (e.g. `~/OneDrive - Synozur/sales-harness/prospects`).
   - `OUTLOOK_FROM` to the mailbox that will own the Drafts.
3. **Fill the ICP definition.** Edit
   `skills/icp-research/icp-definition.md`. Replace every TODO with the
   real Synozur ICP. The Prospector scores against this; vague ICP =
   vague scoring.
4. **Fill the disqualifiers.** Edit
   `skills/icp-research/disqualifiers.md`. Add the competitor list and
   the industry exclusion list.
5. **Initialize the suppression list.** Edit
   `skills/compliance/suppression-list.md` with any internal addresses,
   known competitors, and previously opted-out contacts.
6. **Curate a voice corpus.** Drop 20+ best-performing Synozur sent
   messages into `skills/outbound-voice/corpus/` (gitignored — create
   the folder, the .gitignore already covers it). One message per file,
   strip recipient PII.
7. **Extract Voice DNA.** In a fresh Cowork session, follow
   `skills/outbound-voice/voice-dna-extract.md`. Replace
   `skills/outbound-voice/voice-dna.md` with the output. Commit.
8. **Decide on the Outlook write path.** v1 default is Option C in
   `skills/outlook-ops/SKILL.md` — Composer writes drafts to the MD
   file, a small `tools/sync_drafts.py` (not yet written) materializes
   them into Outlook Drafts via Graph. Until that script exists, the
   operator opens Outlook and creates the Draft manually from the MD
   file. v1 is still useful in that fully-manual mode for the first
   week.

## The daily loop

From a Cowork session opened in the repo root (CLAUDE.md is loaded
automatically):

### Morning — add new prospects

For each new lead:

```
cp prospects/_template.md "$PROSPECT_DIR/firstname-lastname.md"
```

Edit the new file to fill name, email, company, title, linkedin_url,
domain. Save.

Then in Cowork:

```
/run-prospector
```

The Prospector spawns once per `state: new` file. Each writes a
dossier, scores ICP, and transitions to `researched` or `disqualified`.
The session prints a summary table at the end.

### Mid-morning — draft outbound

```
/run-composer
```

For each `state: researched` lead, the Composer writes a draft into the
MD file under `## Draft (pending approval)` and (when the sync script
is wired) saves a Draft to Outlook. State becomes
`draft_pending_approval`.

Open Outlook. Walk the Drafts folder. For each:

- Read the draft. Compare against the dossier in the MD file (the
  reasoning trace is in the MD).
- If good: click Send.
- If needs editing: edit in Outlook and send (your edits are silently
  approved — Cadence will detect the send on the thread).
- If reject: delete the Draft in Outlook AND open the MD file, change
  `state:` back to `researched` and add a `## Notes` line about why.
  The Composer will retry next time you run `/run-composer`.

### Throughout the day — keep the loop moving

```
/run-cadence
```

Detects sends (moves drafts to `sent`), detects replies (classifies and
moves to `replied`), fires templated follow-up steps that pass all
guardrails (auto-saves to Drafts; v1 default is human-still-clicks-send
on these too — auto-send is a future flag, not v1).

Run this at least twice a day. Hourly is fine.

### End of day — review and reset

- Skim any `state: needs_review` files. These hit an error. The
  `## Audit` section will say what.
- Skim any `state: draft_pending_approval` files that have been sitting
  for >24h. They are blocking — either approve or reject.
- Skim any `hold_reason:` set. Decide.
- Skim the daily summary from the last `/run-cadence` — was anything
  auto-paused?

## Weekly hygiene

- Run `/run-prospector --include-dormant` once a week to sweep dormant
  leads for new signals. (Flag not yet implemented in v1; do this
  manually for now by editing dormant files back to `state: new`.)
- Review the campaign reply rate. If trending down, pause and rewrite
  the templates in `skills/cadence-rules/templates/`.
- Review the voice — re-extract `voice-dna.md` if any drafts are
  reading like the model and not like you.

## What success looks like in week 1

- 20 prospects researched with dossiers worth reading
- 10 of those become `draft_pending_approval`
- 5 drafts get sent (you approve in Outlook)
- 1 reply detected and classified correctly
- Zero auto-sends (v1 keeps the human in the loop)

## What to watch for

- **Drafts that read like the templates.** Voice DNA is too thin —
  add 10 more corpus messages and re-extract.
- **ICP scores that don't match your gut.** ICP definition is too
  vague — tighten `icp-definition.md`.
- **Replies that classify wrong** (e.g. an objection misread as
  interest). Add the wording to the classification examples in the
  Cadence agent's prompt and re-test.
- **Drafts referencing signals that aren't real.** Prospector
  hallucinated a Recent Activity bullet. Tighten the
  cite-as-you-go discipline in
  `skills/icp-research/single-command-audit.md`.

## How this maps to Copilot Cowork in tenant

Everything in this repo is in the Claude Cowork format —
`.claude/agents/`, YAML frontmatter, skills as folders, slash commands
in `.claude/commands/`. Microsoft 365 Copilot Cowork shares the same
substrate. The expected port to a tenant-hosted Copilot Cowork run is:

- Agent definitions → Copilot Cowork agent manifests (same
  frontmatter, possibly renamed fields)
- Skills → Copilot Skills (likely 1:1)
- Slash commands → Copilot prompt-launchers
- Prospect MD files in OneDrive → already in the tenant
- Outlook MCP read tools → native in Copilot, no change
- Outlook write path → use Copilot's built-in Outlook actions instead
  of the sync script

We will port once the Cowork manifest spec is confirmed for the M365
side. Until then, this same repo runs in Claude Cowork (and Claude
Code) against the operator's machine.
