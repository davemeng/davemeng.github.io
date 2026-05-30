# File formats inside a client folder

The on-disk formats the orchestrator agent reads and writes inside each
`Clients/<X>/` folder, plus the ingress's dedupe key rules. Kept here as the
single spec so the agent (`agent.yaml`/`system-prompt.md`), the ingress
(Phase 2+), and you (reviewing in cowork) agree exactly. `<date>` is
`YYYY-MM-DD`; `<ts>` is `YYYYMMDDTHHMMSSZ`; `<slug>` is short kebab-case.

## `id` derivation (the dedupe key — used everywhere)

The ingress computes a **deterministic** `id` from the canonical provider
message id, so retries are idempotent:

```
provider_message_id =
  gmail    -> RFC 5322 Message-ID header (preferred) or Gmail API id
  whatsapp -> whatsmeow event message_id (chat_jid + msg_id)
  plaud    -> Plaud recording id

id = first 12 hex chars of sha256(channel + ":" + provider_message_id)
```

`id` is what appears in filenames (`inbox/<ts>-<id>.md`, `outbox/<id>.md`,
`pending-approval/<id>.md`, `feedback/<id>.json`). The ingress dedupes on
`item_id` before invoking the agent; the agent additionally bails fast
(`action: deduped`) if it sees an existing `inbox/<id>.md` or `outbox/<id>.md`
for the same id.

## `inbox/<ts>-<id>.md` — an incoming item

Filename uses `<ts>-<id>` (no `<source>` segment — that's in the front-matter,
no need to duplicate).

```markdown
---
id: 9f3a2b1c4d5e
source: gmail                    # gmail | whatsapp | plaud
provider_message_id: "<Message-ID@acme.com>"
received_at: 2026-05-13T09:14:00Z
client: acme
routed_by: rule                  # rule | classifier
rule_match: sender_domain        # sender_email | sender_domain | sender_phone | plaud_title | plaud_participant — null if routed_by=classifier
routing_rationale: null          # one line if routed_by=classifier, else null
classified_risk: high            # low | med | high
risk_reason: "global-playbook §1 always-material: contract terms (SPA)."
sender: "Jane Counsel <jane.counsel@lawfirm.com>"
subject: "Re: Project Falcon — revised SPA schedule 3"
thread_ref: "<gmail-thread-id>"
status: staged                   # staged | sent | logged | deduped | superseded
ref: pending-approval/9f3a2b1c4d5e.md   # where the resulting draft/decision lives
---

<verbatim body of the email / WhatsApp text / Plaud transcript+summary>
```

Raw `inbox/` bodies are expired after processing per the retention policy
(default: 30 days). Only the distilled `decisions/` entry is kept long-term.

## `pending-approval/<id>.md` — a draft awaiting sign-off

`kind` discriminates what `mode: approve` should do.

```markdown
---
id: 9f3a2b1c4d5e
client: acme
kind: draft_reply                # draft_reply | meeting_followup | config_proposal
created_at: 2026-05-13T09:14:30Z
provider_message_id: "<Message-ID@acme.com>"   # the inbound this answers; survives inbox/ expiry
channel: gmail                   # how the reply would be sent (n/a for config_proposal)
to: "jane.counsel@lawfirm.com"   # n/a for config_proposal
thread_ref: "<gmail-thread-id>"
risk: high
risk_reason: "global-playbook §1 always-material: contract terms (SPA)."
suggested_action: "Send as-is, or edit the draft body below first."
status: pending                  # pending | superseded
# kind: config_proposal adds these:
# target_file: Clients/Acme/playbook.md
# patch: |
#   @@ ... @@
#   <unified diff>
---

## Draft reply

<the proposed email / WhatsApp text — edit before approving if you want>

## Why this draft

<2–4 lines: what changed, what the agent is proposing, what (if anything) it
wasn't sure about, what facts it'd want before sending>
```

Approve by replying `/approve <id>` (WhatsApp/email) or, in the cowork project,
telling Claude "send the proposal in pending-approval/<id>". On approval the
reply is sent via the right MCP, this file moves to `outbox/<id>.md`, and a
`decisions/` entry is appended. Reject with `/reject <id> <reason>` — the file
is marked `status: superseded` and moved to `pending-approval/.rejected/`.

## `outbox/<id>.md` — audit copy of something already sent

Same front-matter as `pending-approval/` plus:

```yaml
sent_at: 2026-05-13T09:18:02Z
sent_via: gmail                                  # gmail | whatsmeow
provider_message_id_out: "<Outbound-Message-ID>" # of the SENT message — the loop-prevention key
auto: false                                       # true if auto-sent without approval
```

`provider_message_id_out` is what the ingress checks to confirm an incoming
provider event isn't echoing one of our own sends — combined with the
"processed" Gmail label / WhatsApp marker, this prevents the agent from
re-processing its own replies.

## `decisions/<date>-<slug>.md` — the append-only ledger

```markdown
---
date: 2026-05-13
client: acme
kind: reply                      # reply | followup | meeting | lesson | route-fix
ref: outbox/9f3a2b1c4d5e.md      # or meetings/... / feedback/... / null
auto: false                      # true if auto-sent without approval
---

**What happened:** Replied to Jane Counsel confirming we accept the revised
wording of SPA Sch. 3 §2 but flagging the indemnity cap in §4 still needs the
client's sign-off.

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
provider_message_id: "<plaud-id>"               # mirrored for dedupe
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

## `feedback/<id>.json` — a labelled correction (the learning loop)

```json
{
  "id": "9f3a2b1c4d5e",
  "ts": "2026-05-13T11:02:00Z",
  "channel": "gmail",
  "client": "acme",
  "ref": "outbox/9f3a2b1c4d5e.md",
  "input_summary": "Counsel email re revised SPA Sch. 3",
  "agent_classification": {
    "client": "acme",
    "risk": "high",
    "routed_by": "rule",
    "rule_match": "sender_email"
  },
  "agent_output_summary": "<one-line summary of what the agent drafted/did>",
  "your_final_output_summary": "<one-line summary of what was actually sent, if edited>",
  "your_action": "approved_with_edits",
  "correction_kind": "tone_edit",
  "diff": "<concise unified diff, or null>",
  "your_reason": null,
  "derived_lesson": "Client wants schedule references written as 'Sch. 3' not 'Schedule 3'."
}
```

Enums:

- `your_action` ∈ `approved | approved_with_edits | rejected | undone |
  re_routed | lesson_rejected`
- `correction_kind` ∈ `tone_edit | material_misclass | routing_miss |
  rule_addition | other` — matches the pattern categories the reflection
  run looks for (see `system-prompt.md` → `mode: reflect`).

Records are summaries + diffs only, never full transcripts/bodies. The
reflection run reads the last 30 days and proposes config edits when the same
`(correction_kind, target topic/field)` appears ≥ 2 times.

## `examples/<slug>.md` — a curated (situation → ideal reply) pair

```markdown
---
client: acme
tags: ["scheduling", "counsel"]   # used for retrieval — see below
added: 2026-05-13
source: outbox/9f3a2b1c4d5e.md    # where this exemplar came from
---

## Situation
<short description of the incoming context>

## Ideal reply
<the reply you'd want sent in this situation>
```

**Retrieval (v1, lightweight):** the drafter scores each example by
`(tag overlap with the inbound's classified tags) + (lexical similarity of the
Situation block to the inbound's snippet)`, and takes the top 2–3. Cap the
example bank at ~30 per client to keep retrieval cheap; the reflection run
prunes stale or redundant entries via a `config_proposal` when needed. (Phase
9, optional: swap lexical similarity for an embedding-based score.)
