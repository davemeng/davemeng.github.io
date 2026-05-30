<!--
  CLIENT CONTEXT FILE — fill this in per client.

  Fastest way to populate it: open this client's existing cowork project and ask:

    "Summarize this engagement so a new assistant could pick it up cold:
     - the client (who they are, what they do)
     - my role and what I've been engaged to do
     - key people on their side and on any counterparty side (names, roles)
     - current status of the deal/work and what's outstanding
     - the important decisions made so far and why
     - how I like replies written for this client (tone, length, sign-off,
       anything to always include or never say)"

  Paste the result below, then trim. The orchestrator agent reads this on every
  item for this client. The agent may also propose edits to this file via the
  reflection run — those come to you as a pending-approval item.
-->

# <Client name> — engagement context

## Who they are
<one short paragraph>

## My role
<what you've been engaged to do; "commercial advisor on ..." etc.>

## Key people
- **<Name>** — <role>, their side. <email/phone if relevant>
- **<Name>** — <role>, their side.
- **<Name>** — <role>, counterparty side.
- (Detailed contact routing lives in `contacts.yaml`; this is just the cast.)

## Current status
<where the deal/work stands; the live workstreams; what's blocked on whom>

## Decisions so far
<the material things already agreed and the reasoning — keep this current; the
append-only ledger in `decisions/` has the full history with dates>

## How to write for this client
- Tone: <e.g. formal / warm-professional / very terse>
- Length: <e.g. short — 3–5 sentences max unless substantive>
- Sign-off: <e.g. "Best, <you>" / matches the thread>
- Always: <e.g. CC the project manager on anything touching timelines>
- Never: <e.g. quote a number without my sign-off / commit to a date>

## Notes / quirks
<anything else that would trip up someone handling this engagement>
