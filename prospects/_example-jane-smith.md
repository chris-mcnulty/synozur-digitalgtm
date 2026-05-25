---
# Identity
name: Jane Smith
email: jane.smith@acme-example.com
company: Acme (Example)
title: VP, Revenue Operations
linkedin_url: https://www.linkedin.com/in/jane-smith-example
domain: acme-example.com
timezone: America/New_York

# State — this file shows the lead at state: awaiting_reply (after step 1 sent)
state: awaiting_reply
state_updated_at: 2026-05-22T14:31:00Z
last_agent: cadence

# Campaign
campaign: outbound-email-v1
lawful_basis: legitimate-interest

# Cadence runtime
touch_count: 1
next_step_at: 2026-05-27T13:00:00Z
thread_id: AAQkADExAMPLE
sent_at: 2026-05-22T14:30:00Z
cadence_template_id: outbound-email-v1

# Compliance flags
do_not_contact: false
email_invalid: false

# Outcomes
icp_score: 8
disqualify_reason: null
hold_reason: null
re_engage_at: null
---

# Jane Smith — VP RevOps, Acme (Example)

> This is an anonymized example showing a prospect MD file mid-flight.
> Real prospect files do not live in the repo. See `_schema.md` for the
> field reference and `_template.md` for the blank starting point.

## Dossier

- Name: Jane Smith
- Title: VP, Revenue Operations (joined Mar 2025; 14 months in role)
- Company: Acme (Example), ~600 employees, B2B SaaS, $80M ARR (estimate)
- Industry: revenue intelligence / sales analytics
- Reports to: CRO
- Source: LinkedIn profile (https://www.linkedin.com/in/jane-smith-example)

## Signals

- Posted on LinkedIn 2026-05-18: argued that "AI SDR" tools are a category
  error and what teams actually need is operator-grade orchestration
  (https://www.linkedin.com/posts/jane-smith-example_post)
- Acme raised a $40M Series C announced 2026-04-29 (TechCrunch link)
- Acme posted 4 GTM-ops roles in the last 14 days (LinkedIn careers)
- No podcast or conference appearances in last 60 days

## ICP Fit

**Score: 8/10.** Center of ICP on role and company size; recent funding
+ hiring + a public post articulating the exact pain we sell into.

## Hook

Her 2026-05-18 LinkedIn post arguing AI SDR is a category error and what
teams need is operator-grade orchestration. That is precisely the
positioning Synozur uses; specific reference is high-signal.

## Prior Touches

- No prior emails from any Synozur address (Outlook search 2026-05-22).
- No prior SharePoint research on Acme.

## Notes

- LinkedIn post comments include a few competitor CEOs — she has a public
  audience in our space.
- Signature in her recent reply elsewhere uses "Jane (she/her)".

## Draft (pending approval) — 2026-05-22T14:00:00Z

**Channel:** email
**To:** jane.smith@acme-example.com
**Subject:** Your post on AI SDR as a category error

Jane — saw your May 18 post on AI SDR being a category error. Agree
on the diagnosis; the part we keep finding harder than expected is the
operator-grade orchestration piece, specifically how to keep three
agents from stepping on each other when they all want to write to the
same lead.

We're working on this with a few RevOps leaders at Acme-sized
companies. Worth a 10-minute call if the next-quarter version of your
post would include "how we actually shipped this"?

— Chris

---
**Reasoning trace:**
- Hook used: 2026-05-18 LinkedIn post arguing AI SDR is a category error
- Voice cues applied: opens with a specific reference + date; em-dash;
  no "Hope this finds you well"; question close that names a concrete
  next deliverable (her next post)
- Compliance checks passed: banned phrases (none), suppression (clean),
  send caps (under)

## Conversation

**2026-05-22T14:31Z — sent (Chris)**
Subject: Your post on AI SDR as a category error
(body identical to Draft above)

## Audit

```
2026-05-21T09:14:00Z [operator] created file, state: new
2026-05-21T09:32:11Z [prospector] state: new → researched (icp_score: 8)
2026-05-22T13:58:42Z [composer] draft written + saved to Drafts; state: researched → draft_pending_approval
2026-05-22T14:31:00Z [cadence] send detected in sent-items; state: draft_pending_approval → sent → awaiting_reply
2026-05-22T14:31:00Z [cadence] next_step_at scheduled for 2026-05-27T13:00:00Z (template outbound-email-v1, step 2)
```
