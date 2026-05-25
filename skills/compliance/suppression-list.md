# Suppression List

Append-only list of email addresses and domains never to contact.

Format: one entry per line. Comments allowed with `#`. Entries are
case-insensitive. Domain entries match all addresses on that domain.

The Cadence agent appends an entry on every opt-out. Removals require an
explicit human operator decision and a PR with the reason.

```
# Format examples (delete these examples before going live):
# example@example.com   # opted out 2026-04-12 via reply
# competitor.com        # competitor — never contact
# @synozur.com          # internal — never contact ourselves
```

# Active suppressions

(empty — populated by operations)
