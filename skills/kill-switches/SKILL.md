---
name: kill-switches
description: The canonical list of kill switches every agent checks at the top of every loop tick. If any trigger is active, the agent stops the relevant work.
---

# Kill Switches

A kill switch is a condition that, when active, **stops an agent from
acting**. They are not warnings. They are not soft signals. They are
hard stops.

## Files in this skill

- `triggers.md` — the exact triggers and their effects. Read at the top
  of every agent loop and immediately before every send.

## The rule

Every agent — Prospector, Composer, Cadence — calls into this skill
before doing any work, and again before any send. If any trigger is
active, the agent reports it, logs to the prospect MD file (or to a
global log), and exits cleanly.

## Why this is its own skill, not inside `compliance/`

Compliance checks gate content (banned phrases, suppression, CAN-SPAM).
Kill switches gate behavior (campaign auto-pause, per-rep send caps,
the global pause flag, error-rate thresholds). They look similar but
they apply at different points in the flow: compliance is per-draft,
kill switches are per-tick.

Keeping them separate makes it harder to accidentally remove a kill
switch when editing compliance, and easier to add new ones without
touching compliance.

## Authoring new kill switches

A new trigger goes into `triggers.md`. Every agent's prompt already
imports this skill, so once the trigger is in the file, all three
agents check it. No code change required.
