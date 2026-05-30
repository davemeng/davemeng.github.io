# Global playbook

Cross-client rules the orchestrator agent reads on **every** item, in addition
to the matched client's `playbook.md`. This file is the **single source of
truth** for the risk-classification rule and the tone defaults —
`system-prompt.md` defers to what's here so the rules don't drift between two
places. Keep this short and high-confidence; new entries should come from the
weekly reflection run (a pattern seen across ≥2 clients) and be approved like
any other change.

## Risk classification rule

Decide LOW / MED / HIGH in this order. First match wins.

1. **Forced HIGH — if ANY of these holds.** No other check needed.
   - The matched client's `contacts.yaml` flags the sender as
     `always_review: true`.
   - The matched client's `playbook.md` flags the topic, the sender, or a phrase
     in the body as always-ask / material.
   - The item touches any of the always-material categories in step 2.

2. **Always-material categories.** Anything about:
   - **Price / fees:** day rate, retainer, success fee, discounts, payment
     terms, milestone payments, late fees.
   - **Scope / timeline / deliverables:** change requests, anything that moves a
     deadline by more than the playbook's slack, scope additions, exclusions.
   - **Contract terms:** NDAs beyond the standard template, MSAs, SOWs, SPAs,
     LOIs, term sheets, indemnities, liability caps, IP ownership, exclusivity,
     non-compete/non-solicit, termination, governing law.
   - **Cap table / financing:** equity, options, valuation, dilution, anti-
     dilution, pre-emption, drag/tag, anything with a number that could appear
     in a financing.
   - **People:** headcount, hiring, redundancies, comp.
   - **Counterparty legal:** anything a counterparty's lawyer sends or is copied
     on (their party only — your own counsel may be in `contacts.yaml`).
   - **External commitments:** anything that commits the client to a meeting,
     call, or deadline with a third party that isn't a simple "yes, that time
     works".

3. **LOW — only if ALL of these hold.**
   - It's a reply within an existing thread to a known contact (in
     `contacts.yaml`).
   - It conveys no new commercial position — it confirms, acknowledges,
     schedules, forwards, or sends an already-approved standard document.
   - It contains no numbers that matter, no contract language, and no scope or
     timeline change.
   - The client's `playbook.md` doesn't flag the topic or the sender as
     always-ask.
   - The classifier is confident (not uncertain about routing, parties, or
     topic).

4. **Otherwise MED.** Default to MED whenever an item isn't clearly LOW and
   isn't covered by a forced-HIGH rule. **When in doubt → MED at minimum, HIGH
   if it could move money or commitments.**

`risk_threshold` in `routing.yaml` gates auto-send: an item rated **strictly
below** the threshold may be auto-sent; at or above it, it stages in
`pending-approval/`. So a client with `risk_threshold: med` auto-sends LOW only
and stages MED + HIGH.

## Tone and form (defaults — client `playbook.md` overrides)

- Professional, concise, no filler. Get to the point in the first sentence.
- Don't over-apologise, don't over-explain, don't speculate on the client's
  behalf about anything commercial.
- Match the thread's register and language.
- Never invent facts, figures, dates, or commitments. If a reply needs a fact
  you don't have, that alone makes it MED at minimum — stage it with a note
  about what's missing.

## Handling meeting transcripts (Plaud)

- A transcript is new context, not something to reply to. Summarise it, extract
  action items / commitments / decisions, and update `decisions/`.
- Any follow-up email or amendment that comes out of a meeting is almost always
  HIGH — stage it. Only auto-send a follow-up if it's purely "sending the
  notes / confirming the next slot" and the client `playbook.md` whitelists it.

## Routing

- If you can't confidently route an item to a client, send it to `triage` —
  never guess.
- A user re-route is a routing signal: log it to `feedback/` so the reflection
  run can propose a `routing.yaml` rule.

## Privacy / data handling

- Route on metadata + a short snippet, not the full body.
- Fetch the full body only when actually drafting or classifying borderline
  items.
- Keep `feedback/<id>.json` as one-line summaries + diffs, not full transcripts.
- Never write secrets or full credentials into any file in a client folder.
