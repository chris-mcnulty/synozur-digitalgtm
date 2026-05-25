---
name: icp-research
description: How to research a prospect end-to-end — pull profile, pull company, surface recent signals, score ICP fit, write a dossier. Used by the Prospector agent.
---

# ICP Research

Use this skill when you need to take a raw prospect (name + company, or a
LinkedIn URL, or just an email address) and produce a structured dossier
that the Composer can write a message against.

## Files in this skill

- `icp-definition.md` — who Synozur sells to. The scoring baseline. Read
  every run.
- `disqualifiers.md` — hard reasons to drop a prospect immediately.
- `single-command-audit.md` — the end-to-end research prompt. Execute it
  for each prospect.

## The procedure

1. Read `icp-definition.md` and `disqualifiers.md`. Hold both in mind
   while researching.
2. Execute the prompt in `single-command-audit.md` against the prospect.
   It will pull profile + company + recent signals via search.
3. Score ICP fit on 1–10 based on `icp-definition.md`. Justify in one
   sentence.
4. Apply `disqualifiers.md`. Any hit → score 0, mark `disqualified`.
5. Write the dossier into the prospect MD file using the structure in
   `skills/prospect-files/SKILL.md`.

## Quality bar

- Every Recent Activity bullet must cite a URL or a tool call. No
  invented signals.
- The Hook field must be one specific signal, not a category. "Posted
  about agent eval frameworks on Nov 14" is good. "Interested in AI" is
  not.
- If the search returns nothing useful in the last 30 days, write
  "No recent signals" and lower the ICP score by 1 — recency matters
  for cold outreach.
