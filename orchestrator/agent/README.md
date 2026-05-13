# Managed Agent — orchestrator brain (Phase 1)

The autonomous "brain": classify → route → act for each inbound item, plus
approve / reject / reflect modes. Runs on the Claude Developer Platform (Managed
Agents) — **a personal Max subscription does not cover this**; see the plan's
"Costs & subscription" section for the Max-only alternative (a self-hosted
`claude -p` runner using this same `system-prompt.md`).

## Files

- `agent.yaml` — portable agent definition: model, MCP servers, input schema,
  success rubric, completion webhook. Map onto the platform's actual config
  format when you create the agent.
- `system-prompt.md` — the system prompt (referenced by `agent.yaml`).

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

Invoke the agent manually (no ingress yet) with a synthetic payload:

```json
{
  "mode": "process",
  "channel": "gmail",
  "item_id": "test-pricing-1",
  "sender": "ceo@acme.io",
  "subject": "Re: Project Falcon — proposed fee",
  "body": "Can you confirm the day rate we discussed? And can we lock it for 6 months?",
  "thread_ref": "test-thread-1",
  "occurred_at": "2026-05-13T09:00:00Z"
}
```

Expect: `Clients/Acme/inbox/...test-pricing-1.md` written, classified **HIGH**
(pricing), and `Clients/Acme/pending-approval/test-pricing-1.md` written with a
draft + a rationale that names the pricing/lock-in as the reason. Then try a
low-risk payload:

```json
{
  "mode": "process",
  "channel": "gmail",
  "item_id": "test-sched-1",
  "sender": "ceo@acme.io",
  "subject": "Re: weekly sync",
  "body": "Tuesday 3pm works for me — see you then.",
  "thread_ref": "test-thread-2",
  "occurred_at": "2026-05-13T09:05:00Z"
}
```

Expect: classified **LOW**; with Acme's `risk_threshold: med` it would auto-send
a brief confirmation — but since Gmail send is wired but you may not want a real
send during testing, point it at a test thread or temporarily set Acme's
`risk_threshold: low` so it stages instead. Confirm the `outbox/` + `decisions/`
(or `pending-approval/`) artifacts and the one-line completion summary.

## v1 scope

One agent, doing everything inline. Splitting into a Haiku classifier sub-agent +
a Sonnet drafter sub-agent is a later optimisation (do it if classification cost
or draft quality warrants). Don't build the sub-agent fan-out until the single
agent loop is proven end-to-end.
