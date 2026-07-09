# LeadRelay

**A self-hosted [n8n](https://n8n.io) workflow that triages inbound leads with a local LLM — no cloud AI, no SaaS automation platform.**

A lead hits a webhook → a **locally-running Ollama model** classifies its urgency
and intent → the sales team gets an instant Telegram alert with a one-line summary
and a routing decision. Everything runs on your own machine: the automation engine
(n8n), the AI (Ollama), and the data never leave the box.

Built to demonstrate that "workflow automation + AI" doesn't have to mean
Zapier + OpenAI. When a client can't send lead data to a third-party platform,
this is the pattern: self-hosted orchestration, self-hosted inference.

---

## The workflow

```
 POST /webhook/lead
        │
        ▼
 ┌─────────────┐   ┌──────────────────────┐   ┌──────────────┐   ┌────────────────┐   ┌──────────┐
 │ Lead Webhook│──►│ Classify w/ Ollama   │──►│ Shape result │──►│ Notify Telegram│──►│ Respond  │
 │  (trigger)  │   │ qwen2.5:3b, JSON-mode │   │ (route rule) │   │ (sales alert)  │   │  (JSON)  │
 └─────────────┘   └──────────────────────┘   └──────────────┘   └────────────────┘   └──────────┘
```

1. **Lead Webhook** — receives `{ name, message }` over HTTP.
2. **Classify with local Ollama** — a schema-constrained call to the local
   Ollama server returns `{ urgency: hot|warm|cold, intent, summary }`. The JSON
   schema is enforced at the sampler level, so the output always parses.
3. **Shape result** — applies a routing rule (`hot → call now`, `warm → follow up
   today`, `cold → nurture queue`).
4. **Notify Telegram** — sends the sales team an instant alert (token via env).
5. **Respond** — returns the structured classification to the caller.

---

## Run it

Requires [Ollama](https://ollama.com) (with `qwen2.5:3b`) and Node.js 18+.

```bash
# 1. self-host n8n (no Docker required)
npm install -g n8n

# 2. give the Telegram node its secrets (optional — the rest works without it)
export TELEGRAM_BOT_TOKEN=...
export TELEGRAM_CHAT_ID=...

# 3. start n8n and import the workflow
n8n import:workflow --input=workflows/lead-router.json
n8n start          # opens http://localhost:5678
```

Activate the workflow in the UI, then fire a lead at it:

```bash
curl -X POST http://localhost:5678/webhook/lead \
  -H "Content-Type: application/json" \
  -d '{"name":"Dana","message":"Our AC died and the office is 30°C — can someone come today?"}'
# → {"name":"Dana","urgency":"hot","intent":"emergency HVAC repair","routed_to":"call now", ...}
```

---

## Why this shape

- **No cloud AI** — classification runs on a local Ollama model, so lead content
  never leaves the network.
- **No SaaS automation lock-in** — n8n is self-hosted; the workflow is a portable
  JSON file you own, not a Zapier account.
- **Deterministic AI output** — schema-enforced JSON means the downstream nodes
  never break on malformed model output.

Pairs naturally with [PrivateRAG](https://github.com/yagaMI-Reverse/privaterag) —
same philosophy (local orchestration + local inference), different job.

---

## Twilio SMS variant

`workflows/lead-router-sms.json` is the same pipeline with an added **Twilio SMS**
node, so the owner gets an instant text (not just Telegram) the moment a lead is
classified. `Shape result` fans out to Telegram **and** Twilio in parallel; both use
`continueOnFail` so a flaky channel never kills the run.

```bash
# extra env for the SMS node (Twilio free trial works — trial sends to verified numbers)
export TWILIO_ACCOUNT_SID=AC...
export TWILIO_AUTH_TOKEN=...
export TWILIO_FROM=+1...      # your Twilio number
export TWILIO_TO=+1...        # owner's mobile

n8n import:workflow --input=workflows/lead-router-sms.json
```

The Twilio node calls the Messages API directly (`POST /2010-04-01/Accounts/{SID}/Messages.json`,
HTTP Basic auth built from the env vars, form-encoded `From`/`To`/`Body`) — no extra
n8n community node required.

---

## HubSpot CRM variant

`workflows/lead-router-hubspot.json` logs every classified lead straight into
**HubSpot CRM** — so the sales team gets a real, searchable contact record, not just
a notification. `Shape result` fans out to HubSpot **and** Telegram in parallel; the
lead's urgency maps to `hs_lead_status` (`hot → NEW`, `warm → OPEN`, `cold → UNQUALIFIED`).

```bash
export HUBSPOT_TOKEN=pat-...          # HubSpot private-app / service-key token
n8n import:workflow --input=workflows/lead-router-hubspot.json
```

The node calls HubSpot's CRM v3 API directly
(`POST /crm/v3/objects/contacts`, Bearer auth) — no community node needed. The
contact's `firstname`, `email`, `phone`, `hs_lead_status`, and a summary line are
written on every run.

---

*Part of the [yagaMI-Reverse](https://github.com/yagaMI-Reverse) portfolio. The demo business and leads are fictional.*
