# Bootstrap Prompt — Copilot Cowork

A self-contained prompt to paste into a Microsoft 365 Copilot Cowork
session to set up the three sales-prospecting skills (Prospector,
Composer, Cadence) in the Synozur tenant, mirroring the Claude Cowork
manifests in this repo.

Run it once, in a fresh Copilot Cowork session, signed in as a Synozur
M365 user with permission to author Cowork agents and skills.

---

## The prompt (copy from here)

```
You are setting up a three-skill sales-prospecting harness for Synozur
inside Microsoft 365 Copilot Cowork. The reference implementation
already exists in the Claude Cowork format at:

  Repo:     https://github.com/chris-mcnulty/synozur-digitalgtm
  Branch:   claude/autonomous-sales-agents-Hcm4Y

Your job is to read the repo and port its three agents and supporting
skills into Copilot Cowork as three operator skills the user can
invoke: /run-prospector, /run-composer, /run-cadence.

# What the harness does

Three interlocking Cowork skills move a prospect from cold lead to
human-approved outbound:

1. PROSPECTOR — researches a single prospect, writes a dossier into
   their markdown file, scores ICP fit, transitions state.
2. COMPOSER — drafts a first-touch or reply message in the firm voice,
   writes it to the prospect markdown file AND saves it as an Outlook
   Draft for human approval.
3. CADENCE — detects sends, detects replies, classifies replies, fires
   pre-approved templated follow-up steps under guardrails, decides
   when to stop.

The three skills never call each other directly. They coordinate
through:
 - State in each prospect's markdown file (one file per lead in a
   OneDrive folder, YAML frontmatter holds `state:`)
 - Shared sub-skills for ICP, voice, cadence rules, compliance, kill
   switches
 - Outlook Drafts as the approval surface — a human reviews and clicks
   Send

# Files to read in the repo

Read these in order. They define the substrate you are porting.

  /CLAUDE.md
      Operator brief. The role this session is about to play. Adopt the
      tone, output style, and state machine described here.

  /.claude/agents/prospector.md
  /.claude/agents/composer.md
  /.claude/agents/cadence.md
      The three agent definitions, in Claude Cowork frontmatter +
      markdown format. Each defines the agent's role, tools, allowed
      skills, model preference, and transition rules.

  /.claude/commands/run-prospector.md
  /.claude/commands/run-composer.md
  /.claude/commands/run-cadence.md
      The three operator-facing slash commands. These are what the
      Synozur operator types to drive the loop. They become the three
      Copilot Cowork skills you create.

  /skills/icp-research/
  /skills/outbound-voice/
  /skills/cadence-rules/
  /skills/compliance/
  /skills/kill-switches/
  /skills/outlook-ops/
  /skills/prospect-files/
      Seven supporting skills the three agents share. Port each as a
      Copilot Cowork skill (or as referenced knowledge sources,
      whichever the tenant allows). Preserve the file structure.

  /prospects/_schema.md
  /prospects/_template.md
  /prospects/_example-jane-smith.md
      The prospect markdown file schema, blank template, and an
      anonymized worked example. The schema is load-bearing — every
      agent reads and writes against it.

  /docs/architecture.md
  /docs/harness-v1.md
      Design rationale and runbook. Background for your decisions; not
      authoritative over the manifests above.

# What to create in Copilot Cowork

1. Three Copilot Cowork skills that map 1:1 to the three slash
   commands:

   - run-prospector  → spawns the Prospector agent over every prospect
                       MD file in state: new
   - run-composer    → spawns the Composer agent over every prospect
                       in state: researched or replied
   - run-cadence     → spawns the Cadence agent over every in-flight
                       prospect

   Each skill should be invokable from Copilot Chat by name and should
   produce the summary output described in the corresponding
   .claude/commands/*.md file.

2. Three Copilot Cowork sub-agents that the skills delegate to:

   - prospector — tools: SharePoint search/read, Outlook email search
                  (read), Web search, Web fetch, file read/write on
                  OneDrive
   - composer   — tools: file read/write on OneDrive, Outlook email
                  search (read), Outlook draft save (write)
   - cadence    — tools: file read/write on OneDrive, Outlook email
                  search (read), Outlook calendar search (read),
                  find meeting availability, optional Outlook send
                  (DISABLED in v1 — Cadence saves drafts only)

   Each sub-agent's system prompt is the body of the corresponding
   .claude/agents/*.md file (everything after the YAML frontmatter).
   Strip Claude-Cowork-specific frontmatter fields that have no
   Copilot Cowork equivalent (model, color); preserve description,
   tools, allowed-skills, and permission semantics.

3. The seven supporting skills as Copilot Cowork knowledge sources or
   skills, referenced by the three sub-agents:

   - icp-research, outbound-voice, cadence-rules, compliance,
     kill-switches, outlook-ops, prospect-files

   Keep filenames and folder structure identical so future updates
   from the repo port cleanly.

# Configuration the human operator must complete after you set up

Do not invent values for these. Output a checklist the operator must
complete before the first real run:

  - Confirm $PROSPECT_DIR points to the chosen OneDrive folder
    (default: ~/OneDrive - Synozur/sales-harness/prospects)
  - Confirm OUTLOOK_FROM is the mailbox owning the Drafts folder
  - Confirm AGENTS_PAUSED=false in the tenant configuration
  - Fill the ICP definition at skills/icp-research/icp-definition.md
    (currently has TODO placeholders)
  - Fill the disqualifier lists at
    skills/icp-research/disqualifiers.md
  - Curate a corpus of 20+ best-performing Synozur sent messages and
    run the extraction prompt at
    skills/outbound-voice/voice-dna-extract.md
  - Replace skills/outbound-voice/voice-dna.md with the extracted
    voice profile (the Composer refuses to draft until the file no
    longer contains the word PLACEHOLDER)
  - Add any internal addresses, competitors, and historical opt-outs
    to skills/compliance/suppression-list.md

# Critical constraints — non-negotiable

These apply to every skill and sub-agent you create. Repeat them in
each skill's instructions.

  1. The Composer never sends. It writes drafts to the prospect
     markdown file AND to the Outlook Drafts folder. A human opens
     Outlook and clicks Send. That is the approval gate.

  2. The Cadence agent may auto-send ONLY templated follow-up steps
     drawn from skills/cadence-rules/templates/, and only if every
     guardrail in skills/kill-switches/triggers.md passes at send
     time. v1 default is to save these to Drafts too — auto-send is a
     future flag, not v1 behavior.

  3. Every agent checks skills/kill-switches/triggers.md at the top
     of every loop tick AND immediately before any send. Triggers
     include AGENTS_PAUSED, per-rep daily send cap, per-domain weekly
     send cap, campaign reply-rate floor, opt-out words, and the
     Voice DNA placeholder check.

  4. The Prospector cites a URL or tool call for every specific
     claim. No invented Recent Activity bullets. No invented numbers.

  5. The Composer applies the banned-phrases list in
     skills/compliance/banned-phrases.md AND the banned-phrases
     section of skills/outbound-voice/voice-dna.md before saving any
     draft. A draft containing a banned phrase fails and redrafts up
     to 3 times, then holds for human review.

  6. The state machine is the only handoff protocol. An agent only
     touches prospects in its allowed input states and only writes
     its allowed output states. See CLAUDE.md for the full state
     diagram.

  7. Every state transition appends a one-line entry to the prospect
     file's ## Audit section, with timestamp, agent name, and the
     transition. The Audit section is append-only.

# What success looks like at the end of this bootstrap

Output a verification checklist the operator can walk through:

  [ ] /run-prospector visible in Copilot Chat skill list
  [ ] /run-composer visible in Copilot Chat skill list
  [ ] /run-cadence visible in Copilot Chat skill list
  [ ] The three sub-agents (prospector, composer, cadence) exist and
      are linked to the three skills
  [ ] All seven supporting skills are accessible from each sub-agent
  [ ] A dry-run of /run-prospector on the example file
      prospects/_example-jane-smith.md reports zero changes
      (the example is already in state: awaiting_reply, which the
      Prospector should skip)
  [ ] A dry-run of /run-composer with the placeholder voice-dna.md
      stops and reports "Voice DNA not extracted yet" — confirming
      the kill switch fires correctly

# Output format for this bootstrap session

When you are done, produce three artifacts:

  1. A summary of what was created (skill IDs, sub-agent IDs,
     knowledge source IDs).
  2. The operator checklist of pre-flight configuration that must be
     completed before the first real run.
  3. The verification checklist above with pass/fail results from
     your dry-runs.

Do NOT run /run-prospector or /run-composer against real prospects
during this bootstrap. The example file is the only allowed test
target, and only as a dry run.

Begin by reading /CLAUDE.md, then the three agent files, then the
seven skill folders. Confirm understanding of the state machine
before creating anything.
```

---

## How to use this prompt

1. Open a fresh Microsoft 365 Copilot Cowork session as a user who has
   permission to author Cowork agents and skills in the Synozur tenant.
2. Make sure Copilot Cowork can read this repo. Either grant it access
   via the GitHub connector, or paste the repo contents in as
   attachments (the three agent files, seven skill folders, and the
   prospect schema are the minimum).
3. Paste the prompt block above (everything between the fences) as the
   first message.
4. Walk the verification checklist Copilot Cowork outputs at the end.
5. Complete the pre-flight configuration items it lists (Voice DNA
   extraction, ICP definition, suppression list, env vars).

## Tweaking the prompt

If the M365 tenant has a different Cowork manifest format than the
Claude Cowork format used in this repo, the prompt deliberately tells
Copilot Cowork to port the substance and strip what doesn't map. If
the port produces lossy results in a specific field, add a clarifying
sentence under "What to create in Copilot Cowork" and re-run in a
fresh session — do not edit the bootstrap mid-flight, since the agent
state would be inconsistent with the prompt.

## Re-running this prompt

If you re-run, Copilot Cowork should detect the existing skills and
sub-agents and reconcile rather than duplicate. If duplication
happens, the bootstrap should output the duplicate IDs in its summary
so the operator can prune by hand.
