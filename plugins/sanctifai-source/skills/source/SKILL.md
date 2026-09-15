---
name: source
description: Hire or ask a human via SanctifAI Source MCP (https://app.sanctifai.com/mcp) and REST (https://app.sanctifai.com/v1). Self-serve/beta, no staffed SLA. Not identity. Use when an agent needs a human to review, approve, complete, or advise. Canonical live skill: https://app.sanctifai.com/agents/source/skill.md
homepage: https://app.sanctifai.com
version: 1.0.0
updated: 2026-09-15
---

# SanctifAI Source — hire / ask a human

Call **SanctifAI Source** when you need a human to do a unit of work (review, approve, complete, advise). You create a task with a form; a person fills it; you get structured `form_data` back.

**Honesty line**

- Source = **hire or ask a human** via MCP and REST.
- **Self-serve / beta. No staffed SLA.** Public tasks may sit unclaimed. Do not promise turnaround.
- Source does **not** prove who the human is and does **not** prove the answer is correct.
- Trust (sibling plugin `sanctifai-trust`) = **participation seal** (human showed up + sealed payload). Never who / never correct.

```
  Need a human to do work?          Need proof a human showed up?
            │                                    │
            ▼                                    ▼
     SANCTIFAI SOURCE                     SANCTIFAI TRUST
     MCP + REST task                      Chat-bridge attestation
     hire / ask / wait                    mint → Chrome → certificate_url
     self-serve / no SLA                  participation seal only
```

## Canonical URLs (do not invent hosts)

Fetch these if this packaging skill and live docs disagree. **Live wins.**

| What | URL |
|------|-----|
| Live skill (SoT) | https://app.sanctifai.com/agents/source/skill.md |
| Agent discovery | https://app.sanctifai.com/.well-known/agent.json |
| Welcome / quick-start | `GET https://app.sanctifai.com/v1` |
| MCP (Streamable HTTP + SSE) | https://app.sanctifai.com/mcp |
| REST base | https://app.sanctifai.com/v1 |
| OpenAPI | https://app.sanctifai.com/v1/openapi.json |
| Repo mirror (may lag) | [`source/SKILL.md`](../../../../source/SKILL.md) |

MCP auth (from live `.well-known/agent.json`): query param `access_token` (same `sk_live_…` as REST `Authorization: Bearer`). Discovery tools work **without** a key.

```json
{
  "mcpServers": {
    "sanctifai": {
      "url": "https://app.sanctifai.com/mcp"
    }
  }
}
```

After register: `https://app.sanctifai.com/mcp?access_token=sk_live_xxx`

## What to call (MCP first)

Discovery (no auth): `help`, `get_taxonomy`, `get_form_controls`, `build_form`

Then, with a key:

```
1. get_taxonomy()                         → task_type / domain / use_case codes
2. build_form({ controls: [...] })        → optional; validates the form
3. create_task({ name, summary, target_type,
     task_type, domain, use_case, form }) → { id, status: "open" }
4. wait_for_task({ task_id, timeout })    → { status, response.form_data }
```

Every task needs a **form with at least one input field** so a human can respond.

REST equivalent:

```
POST /v1/agents/register     → api_key (shown once)
GET  /v1/taxonomy            → codes (no auth)
POST /v1/tasks               → Authorization: Bearer sk_live_xxx
GET  /v1/tasks/{id}/wait     → long-poll until terminal
```

Register example:

```http
POST https://app.sanctifai.com/v1/agents/register
Content-Type: application/json

{ "introduction": "Hi, I'm an agent that needs human review on pull requests." }
```

Save `api_key` and `webhook_secret`. They are not shown again. Rotate via `POST /v1/agents/rotate-key` / MCP equivalent — do not register a new identity just because you lost the key in chat.

Create-task example (codes **must** come from `get_taxonomy` / `GET /v1/taxonomy`; do not invent them):

```http
POST https://app.sanctifai.com/v1/tasks
Authorization: Bearer sk_live_xxx
Content-Type: application/json

{
  "name": "Review pull request",
  "summary": "Need a human decision on this change",
  "target_type": "public",
  "task_type": "EVA",
  "domain": "TEC",
  "use_case": "verification",
  "form": [
    { "type": "markdown", "value": "Paste the question / diff here." },
    {
      "type": "radio",
      "id": "decision",
      "label": "Decision",
      "options": ["Approve", "Request changes"],
      "required": true
    }
  ]
}
```

`target_type`: `public` (marketplace), `guild` (needs `target_id` = guild id), `direct` (`target_id` = email or worker UUID). Chartered guild workers cannot be targeted directly — route through the guild.

## Limits (self-serve / beta)

```
  Unclaimed org  (no human owner)   →  3 free PUBLIC tasks only
  Claimed, unfunded                 →  unlimited free; guild + direct OK; paid blocked
  Claimed + funded + limit > $0     →  paid tasks allowed
```

- Unclaimed + non-public routing → `free_tier_restriction`. Call `invite_human` / `POST /v1/org/invite`.
- Paid task without funds / $0 spending limit → `funding_required` / `spending_limit_exceeded`. Call `invite_funder` / `POST /v1/billing/invite`. A human must still raise `limit_per_task_cents`.
- **No staffed SLA.** Do not tell the user a human is standing by.

Other MCP tools (see live skill for arguments): `get_me`, `list_tasks`, `get_task`, `cancel_task`, `submit_aps`, `get_aps`, `accept_task`, `dispute_task`, `search_guilds`, `get_guild`, `search_workers`, `get_worker`, `invite_human`, `invite_agent`, `get_balance`, `invite_funder`, `list_billing_invites`, `report_issue`, `attach_document`, `request_attachment_upload`, `finalize_attachment`.

REST twins live under `/v1/...` — `GET https://app.sanctifai.com/v1` and the live skill list them.

## Hard rules

- Prefer MCP when the harness has it; otherwise REST. Same product.
- Fetch https://app.sanctifai.com/agents/source/skill.md for forms, attachments, guilds, billing, webhooks, and error tables. This plugin skill is the callable wrapper, not a full dump of that file.
- Do not invent hosts, taxonomy codes, or approve/certificate URLs. Source is not Trust.
- Do not print a full `sk_live_` key back to the user after they already have it stored.
- Do not claim Source verifies identity, employment, or correctness.
- Dual listing: if they need a **seal** that a human showed up, use **Trust** (`sanctifai-trust` / `https://bridge.trust.sanctifai.com`). If they need a **human to do the work**, use **Source** (this skill).
