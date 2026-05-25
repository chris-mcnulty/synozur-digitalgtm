# synozur-digitalgtm

First-generation harness for autonomous sales prospecting using three
interlocking Cowork agents.

- **Prospector** researches a new lead, scores ICP fit, writes a dossier.
- **Composer** drafts a first-touch or reply message in the firm voice,
  saves it to Outlook Drafts, and queues it for human approval.
- **Cadence** detects replies, schedules follow-ups, fires pre-approved
  templated cadence steps, and decides when to stop.

The three agents never call each other. They cowork through:
- **State** in each prospect's markdown file (`prospects/<slug>.md` in OneDrive)
- **Skills** in `skills/` (ICP, voice, cadence, compliance, kill switches)
- **Outlook Drafts** as the human-approval surface — a person reviews and
  clicks Send.

Authored in the [Claude Cowork](https://support.claude.com/en/articles/13345190-get-started-with-claude-cowork)
agent format (`.claude/agents/`, YAML frontmatter + markdown body, skills as
folders). The same manifests are intended to port to Microsoft 365 Copilot
Cowork in the Synozur tenant, since the two share substrate.

## Quick start

1. Copy `.env.example` to `.env`, set `PROSPECT_DIR` to your synced OneDrive
   folder (e.g. `~/OneDrive - Synozur/sales-harness/prospects`).
2. Fill in the placeholders in `skills/outbound-voice/voice-dna.md`,
   `skills/icp-research/icp-definition.md`, and
   `skills/compliance/suppression-list.md`.
3. Drop one or more new prospect MD files into `$PROSPECT_DIR/` using
   `prospects/_template.md` as the starting shape.
4. From a Claude Cowork (or Claude Code) session in the repo root, run:
   - `/run-prospector` — research all `state: new` prospects
   - `/run-composer` — draft outbound for all `state: researched` prospects
   - `/run-cadence` — schedule follow-ups + detect replies for in-flight leads

See [`docs/harness-v1.md`](docs/harness-v1.md) for the full runbook and
[`docs/architecture.md`](docs/architecture.md) for design rationale.

## Status

v1 is a harness, not a product. Outbound goes to Outlook Drafts; a human
clicks Send. No auto-send path is wired. No production CRM is wired —
prospect state lives in markdown files synced via OneDrive.
