# Voice DNA — Extraction Prompt

Run this in a fresh Cowork session with the corpus folder available.

## Inputs

- A folder of 20+ real Synozur messages that got replies. Each as a
  separate file in `skills/outbound-voice/corpus/`. Plain text. One
  message per file. Strip recipient PII (keep first name only).

## Prompt

```
You are extracting the Voice DNA for Synozur outbound. I have given you
{N} messages that all got replies, all written by Synozur reps.

Output a voice profile with these sections:

1. **Tone** — 3 to 5 adjectives. Be specific. "Direct" and "warm" tell me
   nothing. "Blunt, operator-energy, slightly self-deprecating" does.

2. **Sentence rhythm** — average sentence length in words; how often
   fragments appear; how punctuation is used (em-dashes, semicolons,
   parentheticals).

3. **Vocabulary patterns** — at least 10 words that appear repeatedly
   across the corpus and feel distinctive to this voice; at least 10
   words common in B2B outbound that NEVER appear here.

4. **Opening style** — 3 verbatim examples of how this voice opens cold
   emails. Note the common move (signal-first, question-first, etc.).

5. **Closing style** — 3 verbatim examples of how this voice closes.

6. **Banned phrases** — anything in the messages that would feel
   off-brand, plus standard outbound-cliché bans
   ("Hope this finds you well", "Quick question", "Touching base",
   "I came across your profile", etc.).

7. **Voice sample** — a 200-word message in this voice on the topic
   "the cost of running outbound without an eval harness". Use it as a
   verification test. If a reader can tell it was AI-written, the
   extraction is too thin.

Format the output as markdown, ready to drop into
skills/outbound-voice/voice-dna.md verbatim.
```

## After extraction

1. Replace `voice-dna.md` with the output.
2. Read the voice sample at the bottom. Have a teammate read it.
3. If anyone says "that doesn't sound like us" — add 10 more corpus
   messages, re-run.
4. Commit. The diff will show what changed about the voice over time.

## Cadence

Quarterly, or whenever the rolling 30-day reply rate drops by more than
5 percentage points and we suspect voice drift.
