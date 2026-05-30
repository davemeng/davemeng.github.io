# Managed Agent — orchestrator brain (Phase 1)

The autonomous "brain": classify → route → act for each inbound item, plus
approve / reject / reflect modes. Runs on the Claude Developer Platform (Managed
Agents) — **a personal Max subscription does not cover this**; see the plan's
"Costs & subscription" section for the Max-only alternative (a self-hosted
`claude -p` runner using this same `system-prompt.md`).

## Files

- `agent.yaml` — portable agent definition: model, MCP servers, input schema,
  **completion schema**, success rubric, completion webhook. Map onto the
  platform's actual config format when you create the agent.
- `system-prompt.md` — the system prompt (referenced by `agent.yaml`).

`agent.yaml`'s `input_schema` and `completion_schema` are the contract with the
ingress; `system-prompt.md`'s "Completion contract" section must stay aligned
with `completion_schema` — change one, change both.

## To instantiate (Phase 1 steps from the runbook)

1. Confirm you have Managed Agents access on your Anthropic account.
2. Create the agent from `agent.yaml` + `system-prompt.md`.
3. Attach MCP servers: **Google Drive** (service-account creds scoped to
   `Clients/`, EU data region) and **Gmail** (`gmail.modify` scope). Leave
   **whatsmeow** unattached until Phase 4.
4. Upload `FORMATS.md` (from the orchestrator repo root) so the agent can
   reference the on-disk formats, and make sure `Clients/global-playbook.md` and
   `Clients/routing.yaml` exist in Drive.
5. Set `max_turns` (20) and the per-client token caps from `routing.yaml`.
6. Note the agent's invoke endpoint/credentials — the ingress Worker (Phase 2)
   will call it; the agent's completion webhook points back at the Worker's
   `/webhook/agent`.

## Verify (Phase 1)

Use a test client folder + temporary `risk_threshold: low` for Acme so nothing
real gets sent. `item_id` must be 12 hex chars — derive it from the
`provider_message_id` per FORMATS.md's `id` derivation, or just use a stable
placeholder for tests. Invoke the agent manually (no ingress yet):

```json
{
  "mode": "process",
  "channel": "gmail",
  "item_id": "1111aaaa1111",
  "provider_message_id": "<test-pricing-1@example.com>",
  "sender": "ceo@acme.io",
  "subject": "Re: Project Falcon — proposed fee",
  "snippet": "Can you confirm the day rate we discussed?",
  "body_uri": "drive://test/pricing-1.eml",
  "thread_ref": "test-thread-1",
  "occurred_at": "2026-05-13T09:00:00Z",
  "routing_hint": { "client": "acme", "matched_by": "sender_email", "matched_value": "ceo@acme.io" }
}
```

Expect: `Clients/Acme/inbox/<ts>-1111aaaa1111.md` written, classified **HIGH**
(forced by global-playbook §1 — price/fees), and
`Clients/Acme/pending-approval/1111aaaa1111.md` written with `kind: draft_reply`
and a `risk_reason` naming the pricing rule. Completion JSON has
`action: "staged"` and `pending_ref: "pending-approval/1111aaaa1111"`. Then a
low-risk payload:

```json
{
  "mode": "process",
  "channel": "gmail",
  "item_id": "2222bbbb2222",
  "provider_message_id": "<test-sched-1@example.com>",
  "sender": "ceo@acme.io",
  "subject": "Re: weekly sync",
  "snippet": "Tuesday 3pm works for me — see you then.",
  "body_uri": "drive://test/sched-1.eml",
  "thread_ref": "test-thread-2",
  "occurred_at": "2026-05-13T09:05:00Z",
  "routing_hint": { "client": "acme", "matched_by": "sender_email", "matched_value": "ceo@acme.io" }
}
```

Expect: classified **LOW**; with Acme's `risk_threshold: low` it stages instead
of sending (so you can review without a real send). Flip to `med` and re-run to
confirm the auto-send path produces `outbox/` + `decisions/` (`auto: true`).

Idempotency check: invoke either payload twice with the same `item_id`; the
second invocation must return `action: "deduped"` with no new files.

Also exercise the I/O efficiency rules: confirm that with `routing_hint` set,
the agent does not read `routing.yaml` or other clients' folders (look at the
Drive read trace if your platform exposes it).

## v1 scope

One agent, doing everything inline. Splitting into a Haiku classifier sub-agent +
a Sonnet drafter sub-agent is a later optimisation (do it if classification cost
or draft quality warrants). Don't build the sub-agent fan-out until the single
agent loop is proven end-to-end.
