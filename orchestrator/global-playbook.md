# Global playbook

Cross-client rules the orchestrator agent reads on **every** item, in addition to
the matched client's `playbook.md`. Keep this short and high-confidence — these
override nothing client-specific, they only set the floor. New entries here
should come from the weekly reflection run (patterns seen across ≥2 clients) and
be approved like any other change.

## Always treat as MATERIAL (→ stage in pending-approval, never auto-send)

- Anything about price, fees, rates, discounts, payment terms, or milestone
  payments.
- Anything about scope, deliverables, timelines that affect a deadline, or change
  requests.
- Contract terms: NDAs beyond the standard template, MSAs, SOWs, SPAs, term
  sheets, indemnities, liability caps, IP ownership, exclusivity,
  non-compete/non-solicit, termination clauses.
- Equity, options, valuation, cap-table, or anything with a number that could
  appear in a financing.
- Headcount, hiring, redundancies, or comp.
- Anything a counterparty's lawyer sends or is copied on.
- Anything that commits the client to a meeting, call, or deadline with a third
  party that isn't a simple "yes, that time works".
- Anything where you're not sure — default to MATERIAL.

## Safe to auto-send (LOW risk) — only when ALL of these hold

- It's a reply within an existing thread to a known contact (in `contacts.yaml`).
- It conveys no new commercial position — it confirms, acknowledges, schedules,
  forwards, or sends an already-approved standard document.
- It contains no numbers that matter, no contract language, no scope/timeline
  change.
- The client's `playbook.md` doesn't flag the topic or the sender as always-ask.
- The client's `risk_threshold` permits auto-send.

## Tone and form (defaults — client `playbook.md` overrides)

- Professional, concise, no filler. Get to the point in the first sentence.
- Don't over-apologize, don't over-explain, don't speculate on the client's
  behalf about anything commercial.
- Match the thread's register and language.
- Never invent facts, figures, dates, or commitments. If a reply needs a fact you
  don't have, that alone makes it MATERIAL — stage it with a note about what's
  missing.

## Handling meeting transcripts (Plaud)

- A transcript is new context, not something to reply to. Summarize it, extract
  action items / commitments / decisions, and update `decisions/`.
- Any follow-up email or amendment that comes out of a meeting is almost always
  MATERIAL — stage it. Only auto-send a follow-up if it's purely "sending the
  notes / confirming the next slot" and `playbook.md` whitelists it.

## Routing

- If you can't confidently route an item to a client, send it to `triage` —
  never guess.
- A re-route by the user is a routing signal: log it to `feedback/` so the
  reflection run can propose a `routing.yaml` rule.

## Privacy / data handling

- Send the classifier only what it needs: sender + subject + a short snippet.
- Send full bodies/transcripts to the drafter only when actually drafting.
- Keep `feedback/<id>.json` as one-line summaries + diffs, not full transcripts.
- Don't write secrets or full credentials into any file in a client folder.
