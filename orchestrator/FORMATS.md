# File formats inside a client folder

These are the on-disk formats the orchestrator agent reads and writes inside each
`Clients/<X>/` folder. Kept here so the agent definition (Phase 1) and you (when
reviewing in cowork) share one spec. `<id>` is a short stable id derived from the
source message id; `<date>` is `YYYY-MM-DD`; `<slug>` is a short kebab-case
summary.

## `inbox/<ts>-<source>-<id>.md` — an incoming item

`<ts>` is `YYYYMMDDTHHMMSSZ`. `<source>` ∈ `gmail | whatsapp | plaud`.

```markdown
---
id: 9f3a2b
source: gmail
received_at: 2026-05-13T09:14:00Z
client: acme
classified_risk: high            # low | med | high
routed_by: rule                  # rule | classifier
sender: "Jane Counsel <jane.counsel@lawfirm.com>"
subject: "Re: Project Falcon — revised SPA schedule 3"
thread_ref: "<gmail-thread-id>"
status: staged                   # staged | sent | logged | superseded
ref: pending-approval/9f3a2b.md  # where the resulting draft/decision lives
---

<verbatim body of the email / WhatsApp text / Plaud-transcript-derived note>
```

Raw `inbox/` copies are expired after processing per the retention policy — only
the distilled `decisions/` entry is kept long-term.

## `pending-approval/<id>.md` — a draft awaiting sign-off

```markdown
---
id: 9f3a2b
client: acme
created_at: 2026-05-13T09:14:30Z
in_reply_to: inbox/20260513T091400Z-gmail-9f3a2b.md
channel: gmail                   # how the reply would be sent
to: "jane.counsel@lawfirm.com"
risk: high
rationale: "Touches SPA schedule 3 (contract terms) — material per global + client playbook."
suggested_action: "Send the drafted reply as-is, or edit below first."
---

## Draft reply

<the proposed email / WhatsApp text — edit this before approving if you want>

## Why this draft

<2–4 lines: what changed, what the agent is proposing, what (if anything) it
wasn't sure about>
```

Approve by replying `/approve <id>` (WhatsApp/email) or, in the cowork project,
telling Claude "send the proposal in pending-approval/<id>". On send: the reply
goes out via the Gmail/whatsmeow MCP, this file moves to `outbox/<id>.md`, and a
`decisions/` entry is appended. Reject with `/reject <id> <reason>`.

## `outbox/<id>.md` — audit copy of something already sent

Same front-matter as `pending-approval/` plus `sent_at` and `message_id` of the
outbound message (so the ingress can dedupe it and not re-process it).

## `decisions/<date>-<slug>.md` — the append-only ledger

```markdown
---
date: 2026-05-13
client: acme
kind: reply                      # reply | amendment | meeting | lesson | route-fix
ref: outbox/9f3a2b.md
auto: false                      # true if auto-sent without approval
---

**What happened:** Replied to Jane Counsel confirming we accept the revised
wording of SPA schedule 3 §2 but flagging the indemnity cap in §4 still needs
the client's sign-off.

**Why:** Client had already approved the §2 change verbally (see 2026-05-10
meeting note); §4 is unresolved.

**Follow-ups:** Chase client on §4 indemnity cap.
```

## `meetings/<date>-<slug>.md` — a Plaud transcript, processed

```markdown
---
date: 2026-05-13
client: acme
source: plaud
recording_id: "<plaud-id>"
participants: ["<you>", "Acme CEO", "Acme GC"]
title: "Project Falcon — weekly sync"
---

## Summary
<3–8 lines>

## Decisions made in the meeting
- <decision> — also written to decisions/<date>-<slug>.md

## Action items
- [ ] <owner> — <action> — <due>
- ...

## Follow-ups drafted
- pending-approval/<id>.md — email to <person> re <thing>

## Transcript
<full transcript, or a reference if kept elsewhere per retention policy>
```

## `feedback/<id>.json` — a labeled correction (the learning loop)

```json
{
  "id": "9f3a2b",
  "ts": "2026-05-13T11:02:00Z",
  "channel": "gmail",
  "client": "acme",
  "input_summary": "Counsel email re revised SPA schedule 3",
  "agent_classification": { "client": "acme", "risk": "high", "routed_by": "rule" },
  "agent_output": "<draft the agent produced>",
  "your_final_output": "<what you actually sent, if you edited it>",
  "your_action": "approved_with_edits",
  "diff": "<concise draft → final diff, or null>",
  "your_reason": null,
  "derived_lesson": "Client wants schedule references written as 'Sch. 3' not 'Schedule 3'."
}
```

`your_action` ∈ `approved | approved_with_edits | rejected | undone | re_routed |
lesson_rejected`. Records are kept as summaries + diffs, not full transcripts.
The weekly `mode: reflect` run reads recent records, finds patterns recurring
≥2×, and proposes edits to `playbook.md` / `CLAUDE.md` / `routing.yaml` /
`global-playbook.md` / `examples/` — each proposal arrives as a
`pending-approval/` item.

## `examples/<slug>.md` — a curated (situation → ideal reply) pair

```markdown
---
client: acme
tags: ["scheduling", "counsel"]
added: 2026-05-13
source: outbox/9f3a2b.md          # where this exemplar came from
---

## Situation
<short description of the incoming context>

## Ideal reply
<the reply you'd want sent in this situation>
```

The drafter retrieves the closest 2–3 of these by similarity and includes them
in-context when drafting.
