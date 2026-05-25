---
name: outbound-voice
description: How to write outbound messages in the Synozur firm voice. Used by the Composer agent. Anchored on a Voice DNA artifact extracted from real best-performing messages.
---

# Outbound Voice

Every Composer draft must read like it was written by a Synozur human,
not by a model. This skill is the source of truth for that voice.

## Files in this skill

- `voice-dna.md` — the extracted voice profile. Tone, rhythm,
  vocabulary, opens, closes, banned phrases. **This is THE source of
  truth.** If it still has placeholder content, the Composer must stop
  and ask the operator to run the extraction.
- `voice-dna-extract.md` — the prompt that turns a corpus of past
  messages into `voice-dna.md`. Re-run quarterly or whenever reply rate
  drifts.
- `message-templates.md` — the opener, follow-up, and backup frames.
  Use as starting structure only; rewrite every line in voice.
- `examples/` — worked examples of first-touch, reply-warm, and
  reply-objection messages. Reference but do not copy.
- `corpus/` (gitignored) — real past Synozur messages used as the
  extraction corpus. PII; never commit.

## Quality bar

A draft passes the voice test if a reviewer cannot tell whether it was
written by you last quarter or by the model today. If they can tell,
add 10 more real messages to the corpus and re-run
`voice-dna-extract.md`.

## What the Composer does with this skill

1. Open `voice-dna.md`. Confirm it is not a placeholder.
2. Pick the appropriate frame from `message-templates.md`:
   - first touch (no prior thread)
   - reply-warm (positive reply)
   - reply-objection (skeptical reply)
3. Rewrite the frame line-by-line so the result honors voice-dna.md.
4. Scan for banned phrases (see `skills/compliance/banned-phrases.md`
   AND the banned-phrases section of `voice-dna.md`).
5. Save the draft.

## Anti-patterns

- "I hope this finds you well"
- "I came across your profile and was impressed by..."
- "I wanted to reach out because..."
- "Quick question..."
- Any subject line with the word "Quick" or "Touching base"
- Any opener that does not name a specific signal from the dossier
