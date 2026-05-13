# Cross-project orchestrator

An always-on orchestrator that watches Gmail, WhatsApp (via whatsmeow), and Plaud
meeting recordings, routes each new item to the relevant client engagement, and
either acts autonomously on low-risk items or stages a proposal for approval on
material ones. State for each client lives in a cloud-storage folder that the
matching Claude Desktop cowork project reads as its project files.

The full design — architecture, components, costs, privacy levers, phased
rollout, verification, and open questions — is in
[`/root/.claude/plans/i-am-using-cowork-lexical-globe.md`](../../../root/.claude/plans/i-am-using-cowork-lexical-globe.md)
(the approved plan). This directory holds the buildable artifacts.

## What's here now (Phase 0)

```
orchestrator/
├── README.md                     # this file — the build runbook
├── global-playbook.md            # cross-client rules the agent always reads
├── routing.example.yaml          # routing/risk config schema, with examples
└── clients/
    └── _client-template/         # copy this per client into your cloud-storage root
        ├── CLAUDE.md             # client persona / deal context / tone — fill in per client
        ├── playbook.md           # per-client SOPs + what counts as "material"
        ├── contacts.yaml         # senders that matter + Plaud title patterns
        ├── inbox/                # agent drops incoming items here
        ├── meetings/             # Plaud transcripts + extracted action items
        ├── outbox/               # sent replies (audit copy)
        ├── pending-approval/     # drafts / proposed amendments awaiting sign-off
        ├── decisions/            # append-only ledger of what was done and why
        ├── feedback/             # labeled corrections (for the learning loop)
        └── examples/             # curated (situation → ideal reply) pairs
```

Later phases (Managed Agent definition, Cloudflare Worker ingress, the Hetzner
whatsmeow bridge, the Plaud connector, the reflection job) get added under
`orchestrator/agent/`, `orchestrator/ingress/`, `orchestrator/bridge/`, and
`orchestrator/connectors/` as we work through the rollout.

## Build runbook

Steps marked **[you]** require your own accounts/infra and can't be done from a
coding session. Steps marked **[code]** produce artifacts in this repo.

### Phase 0 — skeleton + pilot migration

1. **[you]** Confirm which connectors Claude Desktop cowork supports as a project
   file source. The default assumed here is **Google Workspace Drive** (Workspace
   account, EU data region). If cowork only offers something else (GitHub repo,
   OneDrive, Notion), tell me and we adjust. *This is the gating open question.*
2. **[you]** In your cloud-storage root, create a `Clients/` folder and copy
   `clients/_client-template/` into it as `Clients/_Client Template/`.
3. **[you]** Pick one real engagement as the pilot (referred to below as `Acme`).
   Copy `Clients/_Client Template/` → `Clients/Acme/`.
4. **[you]** In the Acme cowork project, run the two prompts noted at the top of
   `CLAUDE.md` and `playbook.md` to self-extract the context and the operating
   rules; paste the results in. Fill `contacts.yaml`.
5. **[you]** Connect the `Clients/Acme/` folder to the Acme cowork project as a
   file source.
6. **[you]** Create a Google service account (or equivalent) that the future
   Managed Agent will use; share `Clients/` to it with edit access. Keep the key
   safe — it does **not** go in this repo.
7. **Verify:** drop a test file `Clients/Acme/inbox/2026-05-08-test.md`; open the
   Acme cowork project; confirm it appears as a project file.

### Phase 1 — Managed Agent core  *(next coding session)*

8. **[you]** Confirm you have access to the Claude Developer Platform / Managed
   Agents (note: a personal Max subscription does **not** cover this — see the
   "Costs & subscription" section of the plan; the Max-only alternative is a
   self-hosted `claude -p` runner, which is a different build).
9. **[code]** Add `orchestrator/agent/` — system prompt, MCP server config
   (Google Drive + Gmail to start), input schema, outcome/rubric, single-agent v1.
10. **[you]** Create the Managed Agent in your account from that definition; wire
    the Drive + Gmail MCP servers.
11. **Verify:** invoke the agent manually with a synthetic pricing-email payload;
    confirm it writes `inbox/<id>.md` and stages `pending-approval/<id>.md` with a
    sensible rationale; a synthetic "confirming Tuesday 3pm" payload classifies
    low-risk.

### Phase 2 — Gmail ingress

12. **[you]** GCP project: enable Gmail API + Pub/Sub; `users.watch` on INBOX →
    topic → push subscription. OAuth consent for `gmail.modify` (testing mode is
    fine for solo use).
13. **[code]** Add `orchestrator/ingress/` — Cloudflare Worker: `/webhook/gmail`
    (verify push, pull history deltas, dedupe by `messageId`, invoke agent),
    `/webhook/agent` (completion → email notification). KV for `historyId` +
    processed IDs.
14. **[you]** Deploy the Worker; set secrets (`wrangler secret put ...`).
15. **Verify:** send a real email from a configured sender; within ~60s expect
    `pending-approval/<id>.md` in Drive + an email ping.

### Phase 3 — routing + auto-send

16. **[code]** `routing.yaml` (from `routing.example.yaml`), per-client
    `playbook.md`, risk threshold, low-risk auto-send, `/approve` handling in the
    Worker.
17. **[you]** Onboard 2–3 clients (repeat Phase 0 steps 3–5 per client; add each
    to `routing.yaml`).
18. **Verify:** low-risk pattern → direct send + notification, no
    pending-approval. Pricing email → pending-approval + ping; `/approve <id>` →
    reply sent, file moved to `outbox/`, `decisions/` appended. Flip a client's
    `risk_threshold` to `high`; confirm the low-risk message now stages.

### Phase 4 — WhatsApp

19. **[you]** Provision a Hetzner CX-series VM in an EU datacenter; hardened
    Debian, LUKS full-disk encryption, SSH keys only + fail2ban.
20. **[you]** Decide on a WhatsApp number (a dedicated business number is
    recommended — whatsmeow's multi-device protocol carries a ban risk on
    personal accounts).
21. **[code]** Add `orchestrator/bridge/` — Go process: open whatsmeow session,
    subscribe to `*events.Message`, POST relevant messages to the ingress; also
    serve the whatsmeow MCP over token-gated HTTPS/SSE for the agent to send
    replies.
22. **[you]** Deploy the bridge to the VM; scan the WhatsApp QR; attach the
    remote whatsmeow MCP to the Managed Agent; wire `/webhook/whatsapp`.
23. **Verify:** send a WhatsApp from a configured number; bridge fires
    `/webhook/whatsapp`; agent processes; low-risk reply arrives via whatsmeow;
    the cloud agent can reach the remote whatsmeow MCP.

### Phase 5 — Plaud

24. **[you]** Confirm what Plaud exposes: native webhook/API → email-forward →
    Notion/Drive sync (in that order of preference).
25. **[code]** Add `orchestrator/connectors/plaud/` matching whatever Plaud
    offers; wire `/webhook/plaud`; add `meetings/` handling + `plaud_title_patterns`
    routing. Transcripts default to `pending-approval/`.
26. **Verify:** record a short test meeting whose title matches an Acme
    `plaud_title_pattern`; confirm the connector fires, `meetings/<date>-<slug>.md`
    is written with summary + action items, follow-up draft lands in
    `pending-approval/`. A non-matching meeting lands in `Triage`.

### Phase 6 — feedback capture

27. **[code]** On every `/approve`, `/reject`, `/undo`, edit-then-send (from
    cowork), and re-route, write `feedback/<id>.json` (agent input, agent output,
    your final output, your action, diff, reason, derived lesson). No behavior
    change yet — collect for a couple of weeks.
28. **Verify:** approve one draft unchanged, edit-then-send another, `/reject` a
    third with a reason; confirm three `feedback/*.json` records with the right
    `your_action`, `diff`, `your_reason`.

### Phase 7 — reflection & few-shot

29. **[code]** `mode: reflect` in the agent; weekly cron in the ingress;
    example-bank retrieval in the drafter; `global-playbook.md` updates from
    cross-client patterns; learning metrics in the digest. Proposed edits flow
    through `pending-approval/`.
30. **Verify:** seed ~10 synthetic `feedback/` records with a repeated pattern;
    run `mode: reflect`; confirm a `pending-approval/` proposal adds the rules to
    `playbook.md`; approve it; confirm `playbook.md` updated, `decisions/` entry
    logged, a later test item handled per the new rule; reject a different
    proposal with a reason; confirm it logs a `feedback/` record and doesn't
    reappear.

### Phase 8 — polish

31. **[code]** Daily digest email; `decisions/`/`meetings/` search; WhatsApp
    `/approve` one-tap; Drive reconcile job for cowork-initiated sends; per-client
    monthly token caps + daily spend alert.

## Secrets — never commit

Gmail refresh token, Anthropic API key, whatsmeow session DB, Drive
service-account key, remote-MCP bearer tokens, Cloudflare API token. Keep these
in Cloudflare Worker secrets / the VM's encrypted secret store. This repo holds
**only** code, config schemas, and templates.

## Loop-prevention reminder

The agent's own outbound replies must not retrigger inbox processing: dedupe on
Gmail `messageId` / WhatsApp message ID, and tag outbound mail (a Gmail label)
and WhatsApp messages (a marker) so the ingress skips them — including
cowork-initiated sends.
