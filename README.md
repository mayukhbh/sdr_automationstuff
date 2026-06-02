# Sales Automation Workflows (n8n)

Two importable [n8n](https://n8n.io) workflows that automate top-of-funnel research and dormant-pipeline reactivation. Both assume a **CRM as the source/trigger** and use **Claude** for generation.

| Workflow | Trigger | Output |
|---|---|---|
| **Meeting Prep Assistant** | Meeting booked (CRM webhook) | One-page prospect brief posted to Slack |
| **Lost Deal Reactivation Engine** | Daily schedule | AI re-engagement draft + SDR task on closed-lost accounts showing new buying signals |

---

## Contents

- [Workflow 1 — Meeting Prep Assistant](#workflow-1--meeting-prep-assistant)
- [Workflow 2 — Lost Deal Reactivation Engine](#workflow-2--lost-deal-reactivation-engine)
- [Setup](#setup)
- [Credentials](#credentials)
- [Data you must supply](#data-you-must-supply)
- [Manual configuration](#manual-configuration)
- [Caveats](#caveats)

---

## Workflow 1 — Meeting Prep Assistant

Saves manual pre-call research time by assembling a brief automatically when a meeting is booked.

```
Meeting booked (CRM webhook)
        │
Extract attendee info
        │
Enrich company & contact   ← 1 call: person + firmographics + tech stack + funding
        │
Recent news                ← optional; delete if enrichment provider returns news
        │
Assemble prompt (Code)
        │
Generate brief (Claude)
        │
Post to Slack
```

**Brief sections:** Company Overview · Likely Pains · Talking Points · Suggested Questions.

`meeting_prep_assistant.json` — 7 nodes.

---

## Workflow 2 — Lost Deal Reactivation Engine

Turns closed-lost accounts back into pipeline by watching for buying signals and prompting the SDR when one appears.

```
Daily schedule (07:00)
        │
Get closed-lost accounts (CRM)
        │
Get company signals        ← 1 call: funding + hiring + leadership + M&A (dated event feed)
        │
Detect new signals (Code)  ← filters by category, captures event IDs for de-dup
        │
   New signal? ── no ──► (stop)
        │ yes
Draft re-engagement (Claude)
        │
Create SDR task (CRM)
```

`lost_deal_reactivation_engine.json` — 6 nodes.

---

## Setup

1. **Import** — in n8n: *Workflows → Import from File* → select each `.json`.
2. **Create credentials** — see [Credentials](#credentials). Open each red-flagged node and attach the matching credential.
3. **Wire the source** — point your CRM/calendar automation at the WF1 webhook URL (shown on the Webhook node once the workflow is saved); set the WF2 closed-lost filter to match your CRM.
4. **Test** — run each workflow manually with sample data before activating.
5. **Activate** — toggle each workflow on.

> Default LLM model is `claude-3-5-sonnet-20241022`. Update it in the two Claude nodes to your preferred current model.

---

## Credentials

Every node currently points at a `REPLACE_WITH_CREDENTIAL_ID` placeholder. Create these in n8n and attach them:

| # | Credential | Type / header | Used in |
|---|---|---|---|
| 1 | Anthropic API | HTTP Header Auth — `x-api-key` | both |
| 2 | CRM API *(HubSpot Private App shown)* | HTTP Header Auth — `Authorization: Bearer …` | WF2 |
| 3 | Enrichment API *(Apollo / PDL / Clearbit / Crustdata)* | HTTP Header Auth — provider key | WF1 |
| 4 | Signal API *(Autobound / PredictLeads / Crustdata)* | HTTP Header Auth — provider key | WF2 |
| 5 | Slack | Bot token, scope `chat:write` | WF1 |
| 6 | News API *(optional)* | HTTP Header Auth | WF1 (only if `Recent News` kept) |

**Minimum: 5 paid data/AI accounts** (4 if you drop the news node and your enrichment provider returns news).

---

## Data you must supply

**WF1 — webhook payload.** Your CRM/calendar must `POST` this JSON to the webhook URL:

```json
{
  "attendee_email": "jane@acme.com",
  "attendee_name": "Jane Doe",
  "company_name": "Acme Inc",
  "company_domain": "acme.com",
  "meeting_start_time": "2026-06-10T15:00:00Z",
  "owner_slack_channel": "C0123456789"
}
```

**WF2 — closed-lost source.** Must return one item per account with at least: `account_id` (or `id`), `name`, `domain`, and an owner id. Edit the search filter if your closed-lost stage isn't `closed_lost`.

---

## Manual configuration

These are **adapt**, not just plug-and-play:

- **Enrichment endpoint/body (WF1)** — Apollo `people/match` is shown. Confirm your provider's actual endpoint, request shape, and auth header.
- **Signal response path (WF2)** — the `Detect New Signals` node reads `res.events || res.signals || res.data || res.results`. Set it to your provider's real array path and confirm each event's `type` / `date` / `id` field names.
- **De-duplication (WF2)** — current logic is **stateless** (flags signals within `WINDOW_DAYS`). For true cross-run dedup, persist `eventIds` to a store (Google Sheet, n8n Data Table, or Postgres) and check against it before drafting. Without this, a lingering signal can re-fire daily.
- **Slack fallback** — hardcode a default channel id if `owner_slack_channel` may be empty.
- **Rate limiting (WF2)** — for hundreds of accounts hitting a metered API, add a Loop / Split-in-Batches node with a wait.

---

## Caveats

- **Provider coverage** — enrichment/signal vendors skew US/Western. Run a match-rate test on a sample of your real target accounts before committing to a paid tier.
- **Placeholder endpoints** — the enrichment/signal URLs are realistic placeholders, not verified-current API contracts. Check each vendor's live docs for exact paths and auth before going live.
- **Status** — both workflows ship **inactive**; activate only after a successful manual test run.
