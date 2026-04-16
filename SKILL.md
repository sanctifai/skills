---
name: sanctifai
description: >-
  Human-in-the-loop skill for AI agents. Use when your agent needs humans to
  review, approve, or complete a task. Provides REST API and MCP (Model Context
  Protocol) integration with long-polling and webhooks. No server required —
  agents self-register and get responses back asynchronously.
---

# SanctifAI: Human-in-the-Loop for AI Agents

> **Base URL:** `https://app.sanctifai.com/v1`

You're an AI agent that needs human input. SanctifAI gives you an API to ask humans questions and get structured responses back. Register once, create tasks, and either wait for completion or receive webhooks when humans respond.

---

## Prerequisites

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  WHAT YOU NEED                                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ✓ Ability to make HTTP requests       That's it.                           │
│                                                                             │
│  ✗ No server required                  Use long-poll to wait for responses  │
│  ✗ No pre-registration                 Sign up via API when you need it     │
│  ✗ No human setup                      Fully self-service for agents        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Integration Paths

SanctifAI supports two integration styles. Choose based on your runtime:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  INTEGRATION PATHS                                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  MCP (Model Context Protocol)          REST API                             │
│  ──────────────────────────           ────────                              │
│  Best for: Claude, MCP-native agents  Best for: any HTTP client             │
│                                                                             │
│  Endpoint: POST /mcp                  Endpoint: https://app.sanctifai.com   │
│  Auth: ?access_token=sk_xxx           Auth: Authorization: Bearer sk_xxx    │
│  Protocol: Streamable HTTP + SSE      Protocol: Standard HTTP/JSON          │
│                                                                             │
│  Tools exposed directly to model      You call endpoints manually           │
│  Real-time task status via SSE        Long-poll /v1/tasks/{id}/wait         │
│  Idempotency key support built-in     Pass idempotency_key in body          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## MCP Server

### Connection

Add SanctifAI to your MCP client configuration:

```json
{
  "mcpServers": {
    "sanctifai": {
      "url": "https://app.sanctifai.com/mcp?access_token=sk_live_xxx"
    }
  }
}
```

**Protocol:** Streamable HTTP transport with SSE for real-time notifications. The `access_token` query parameter carries your API key — the same `sk_live_xxx` you get from registration.

**No auth required for discovery tools** — `get_taxonomy`, `get_form_controls`, and `build_form` work without a key.

<!-- GENERATED:TOOLS:START -->
### MCP Tools Reference

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  DISCOVERY (no authentication required)                                        │
├──────────────────────┬──────────────────────────────────────────────────────────┤
│  get_taxonomy        │ Get available task types, domains, and use cases. Call  │
│                      │ this before creating a task to know which task_type,    │
│                      │ domain, and use_case codes to use.                      │
├──────────────────────┬──────────────────────────────────────────────────────────┤
│  get_form_controls   │ Get available form control types and their schemas. Call│
│                      │ this to understand what form elements you can use when  │
│                      │ creating a task.                                        │
├──────────────────────┬──────────────────────────────────────────────────────────┤
│  build_form          │ Validate and normalize a form definition before creating│
│                      │ a task. Returns the normalized form or validation       │
│                      │ errors.                                                 │
└──────────────────────┴──────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────┐
│  AGENT (authentication required)                                               │
├──────────────────────┬──────────────────────────────────────────────────────────┤
│  get_me              │ Get your agent profile, organization info, and task     │
│                      │ statistics. Use this to verify your identity and see how│
│                      │ many tasks you have in each status.                     │
└──────────────────────┴──────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────┐
│  TASKS (authentication required)                                               │
├──────────────────────┬──────────────────────────────────────────────────────────┤
│  create_task         │ Create a new task for humans to complete. Every task    │
│                      │ requires a form with at least one field so humans can   │
│                      │ respond. Use get_taxonomy first to discover valid       │
│                      │ task_type, domain, and use_case codes. Use build_form to│
│                      │ validate your form before submitting. For direct tasks, │
│                      │ target_id accepts either an email address or a worker   │
│                      │ UUID. Chartered guild workers cannot be targeted        │
│                      │ directly — tasks must be routed through their guild.    │
│                      │ ROUTING RESTRICTIONS: Unclaimed organizations (no human │
│                      │ owner) can only create public free tasks — guild and    │
│                      │ direct routing require a claimed org (POST              │
│                      │ /v1/org/invite). Paid tasks require a funded wallet and │
│                      │ a spending limit > $0 set by a human administrator.     │
│                      │ Example form:                                           │
│                      │ [{type:"title",value:"Review"},{type:"radio",id:"decisio│
│                      │ Call get_form_controls for all control types.           │
├──────────────────────┬──────────────────────────────────────────────────────────┤
│  list_tasks          │ List tasks you have created, optionally filtered by     │
│                      │ status. Returns paginated results with task details and │
│                      │ response data.                                          │
├──────────────────────┬──────────────────────────────────────────────────────────┤
│  get_task            │ Get a specific task by ID. Returns full task details    │
│                      │ including response data if completed. Includes          │
│                      │ has_open_issue (boolean) and issues array if the worker │
│                      │ has reported any problems.                              │
├──────────────────────┬──────────────────────────────────────────────────────────┤
│  cancel_task         │ Cancel a task. Only tasks that have not been claimed can│
│                      │ be cancelled. If the task has escrowed funds, they will │
│                      │ be refunded.                                            │
├──────────────────────┬──────────────────────────────────────────────────────────┤
│  wait_for_task       │ Wait for a task to be completed or cancelled. Blocks    │
│                      │ until the task reaches a terminal state or the timeout  │
│                      │ is reached. Returns the task with a timed_out flag.     │
├──────────────────────┬──────────────────────────────────────────────────────────┤
│  submit_aps          │ Submit an Agentic Promoter Score (APS) for a completed  │
│                      │ task. APS is a 0–10 satisfaction rating for the worker's│
│                      │ performance. Must be submitted within 48 hours of task  │
│                      │ completion — after that, a default score of 10 is used. │
│                      │ Idempotent: submitting again updates the existing score.│
├──────────────────────┬──────────────────────────────────────────────────────────┤
│  get_aps             │ Get the APS (Agentic Promoter Score) feedback you       │
│                      │ submitted for a completed task. Returns the score,      │
│                      │ notes, and submission timestamp, or a not-yet-submitted │
│                      │ indicator.                                              │
├──────────────────────┬──────────────────────────────────────────────────────────┤
│  accept_task         │ Accept a completed task and submit your APS (Agentic    │
│                      │ Promoter Score). Must be called within 48 hours of task │
│                      │ completion — after that the task is auto-accepted with  │
│                      │ no APS. Requires aps_score (0-10). Optional notes for   │
│                      │ the worker. Once accepted, the task cannot be disputed. │
├──────────────────────┬──────────────────────────────────────────────────────────┤
│  dispute_task        │ Dispute a completed task you are not satisfied with.    │
│                      │ Must be called within 48 hours of task completion —     │
│                      │ after that the task is auto-accepted. Requires a reason │
│                      │ explaining what was unsatisfactory. Once disputed, the  │
│                      │ task enters dispute resolution and cannot be accepted.  │
│                      │ Do not dispute unless the worker failed to deliver what │
│                      │ was requested.                                          │
└──────────────────────┴──────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────┐
│  GUILDS (authentication required)                                              │
├──────────────────────┬──────────────────────────────────────────────────────────┤
│  search_guilds       │ Search all guilds. Filter by guild_type, domain,        │
│                      │ languages, country, certifications, minimum APS score,  │
│                      │ or minimum member count. Returns guild profile and      │
│                      │ reputation summary per result. Supports cursor-based    │
│                      │ pagination via limit and cursor parameters.             │
├──────────────────────┬──────────────────────────────────────────────────────────┤
│  get_guild           │ Get full guild details including profile and reputation.│
│                      │ Returns name, summary, description, type, member count, │
│                      │ task types, domains, chartered profile fields (if       │
│                      │ applicable), and aggregated reputation stats.           │
└──────────────────────┴──────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────┐
│  WORKERS (authentication required)                                             │
├──────────────────────┬──────────────────────────────────────────────────────────┤
│  get_worker          │ Get full worker profile and reputation in one call.     │
│                      │ Returns profile data (bio, skills, languages, country,  │
│                      │ timezone, availability, rate range, experience,         │
│                      │ education, certifications, job history, guild           │
│                      │ memberships) plus complete reputation stats (APS, task  │
│                      │ volume, disputes). No PII (name, email, photo) or payout│
│                      │ data is included.                                       │
├──────────────────────┬──────────────────────────────────────────────────────────┤
│  search_workers      │ Search workers by profile and reputation criteria.      │
│                      │ Profile filters: skills (array overlap), languages      │
│                      │ (array overlap), country, min_availability (hours/week),│
│                      │ max_rate_cents (hourly_rate_min ≤ this value),          │
│                      │ min_experience (years), worker_type (freelancer or      │
│                      │ chartered). Reputation filters: min_aps, domain,        │
│                      │ task_type, min_tasks. Scope filter: guild_id (restrict  │
│                      │ to guild members). Returns profile summary and          │
│                      │ reputation stats per result. Supports cursor-based      │
│                      │ pagination via limit and cursor params; response        │
│                      │ includes next_cursor.                                   │
└──────────────────────┴──────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────┐
│  INVITES (authentication required)                                             │
├──────────────────────┬──────────────────────────────────────────────────────────┤
│  invite_human        │ Send an email invite to a human to join your            │
│                      │ organization. The human will receive an email with a    │
│                      │ link to accept the invitation.                          │
├──────────────────────┬──────────────────────────────────────────────────────────┤
│  invite_agent        │ Create a new API key for another AI agent in the same   │
│                      │ organization. Returns the API key and webhook secret for│
│                      │ the new agent.                                          │
└──────────────────────┴──────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────┐
│  BILLING (authentication required)                                             │
├──────────────────────┬──────────────────────────────────────────────────────────┤
│  get_balance         │ Get your organization's wallet balance and spending     │
│                      │ limits.
Returns funded status, available/locked         │
│                      │ balances, and per-agent task spending limits.           │
│                      │ funded=false means the wallet has never been topped up —│
│                      │ paid tasks will be blocked. limit_per_task_cents=0 also │
│                      │ blocks paid tasks even when funded — a human must raise │
│                      │ the limit. Use to diagnose funding_required or          │
│                      │ spending_limit_exceeded errors.                         │
├──────────────────────┬──────────────────────────────────────────────────────────┤
│  invite_funder       │ Send a billing invite to a human administrator. They    │
│                      │ will be able to create an account and fund your         │
│                      │ organization wallet. Funding unlocks paid tasks         │
│                      │ (price_cents > 0) across all routing types (public,     │
│                      │ guild, direct). Note: after funding, a human must also  │
│                      │ raise your per-agent spending limit above $0 before paid│
│                      │ tasks are allowed. Use this when you receive a          │
│                      │ funding_required error.                                 │
├──────────────────────┬──────────────────────────────────────────────────────────┤
│  list_billing_invites│ List billing invites you have sent to human             │
│                      │ administrators. Shows invite status (pending, redeemed, │
│                      │ expired) and the invite URL. Use this to check whether a│
│                      │ previously sent invite has been acted on.               │
└──────────────────────┴──────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────┐
│  ISSUES (authentication required)                                              │
├──────────────────────┬──────────────────────────────────────────────────────────┤
│  report_issue        │ Report a bug, feature request, or question about the    │
│                      │ platform. Use this to contact the SanctifAI team — NOT  │
│                      │ for scoring workers or tasks.                           │
└──────────────────────┴──────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────┐
│  ATTACHMENTS (authentication required)                                         │
├──────────────────────┬──────────────────────────────────────────────────────────┤
│  attach_document     │ Attach a document to a task, either as uploaded bytes OR│
│                      │ by referencing any http(s) URL. Use content_base64 to   │
│                      │ upload bytes (accepted file types: PDF, PNG, JPG/JPEG,  │
│                      │ WEBP, TXT, CSV; max 5 MB per file; max 20 MB total per  │
│                      │ task; no data URI prefix). Use file_url to reference any│
│                      │ http(s) URL — the server stores it verbatim and NEVER   │
│                      │ fetches it; the worker's browser loads it when viewing  │
│                      │ the task, using the worker's own session for            │
│                      │ destinations that require auth (Google Drive, Notion,   │
│                      │ Figma, Dropbox, etc.). Any URL shape is accepted: direct│
│                      │ file URLs, folder links, document pages, landing pages, │
│                      │ cloud-storage share links — we do not inspect path      │
│                      │ structure or MIME type. Destination-side access control │
│                      │ (e.g. 'Anyone with the link') is the caller's           │
│                      │ responsibility. Hard restrictions for file_url: under   │
│                      │ 2048 chars, no embedded credentials, http(s) only,      │
│                      │ hostname must resolve to a public IP (private / loopback│
│                      │ / link-local rejected). Exactly one of content_base64 or│
│                      │ file_url must be provided. The task must belong to your │
│                      │ agent.                                                  │
└──────────────────────┴──────────────────────────────────────────────────────────┘
```
<!-- GENERATED:TOOLS:END -->

### MCP Quick Start

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  MCP WORKFLOW                                                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Step 1: Discover taxonomy (once)                                           │
│  ──────────────────────────────                                             │
│  get_taxonomy()  →  valid task_type, domain, use_case codes                │
│                                                                             │
│  Step 2: Build your form (optional but recommended)                         │
│  ──────────────────────────────────────────────────                         │
│  build_form({ controls: [...] })  →  normalized controls                   │
│                                                                             │
│  Step 3: Create a task                                                      │
│  ─────────────────────                                                      │
│  create_task({                                                              │
│    name, summary, target_type,                                              │
│    task_type, domain, use_case,   ← required, from get_taxonomy            │
│    form: [...]                                                              │
│  })  →  { id: "task_xxx", status: "open" }                                 │
│                                                                             │
│  Step 3b: Attach documents (optional)                                       │
│  ─────────────────────────────────────                                      │
│  attach_document({                                                          │
│    task_id: "task_xxx",                                                     │
│    file_name: "report.pdf",                                                 │
│    mime_type: "application/pdf",                                            │
│    content_base64: "<base64-encoded file bytes>"                            │
│  })  →  { id: "att_yyy", file_name: "report.pdf", size_bytes: 45000 }      │
│                                                                             │
│  Step 4: Wait for the human                                                 │
│  ──────────────────────────                                                 │
│  wait_for_task({ task_id, timeout: 120 })                                  │
│  →  { status: "completed", response: { form_data: {...} } }                │
│                                                                             │
│  Step 5: Rate the worker (optional, within 48h)                             │
│  ─────────────────────────────────────────────                              │
│  submit_aps({ task_id, aps_score: 9 })                                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## REST API

### Quick Start

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  AGENT ONBOARDING (One-time setup)                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Step 1                             You now have an API key!               │
│   ──────────────────────             ──────────────────────                 │
│   POST /v1/agents/register    ────►  Bearer sk_live_xxx                     │
│                                      (save it - shown only once!)           │
│   "Hi, I'm Claude"                                                          │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  CREATING WORK                                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Step 1               Step 2               Step 3               Step 4    │
│   ──────────           ──────────           ──────────           ──────── │
│   GET /v1/         ──► POST /v1/tasks   ──► GET /v1/tasks/   ──► Human     │
│   taxonomy             (with codes          {id}/wait            response  │
│                         from above)         (blocks until        returned  │
│   Pick task_type,                            human completes)   to you     │
│   domain, use_case                                                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Account Lifecycle

Every agent starts **unclaimed**. A human must claim your organization to unlock full routing, and fund your wallet to unlock paid tasks.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  PERMISSION MATRIX                                                          │
├───────────────────────────┬────────┬────────┬────────┬────────┬────────────┤
│  State                    │ Public │ Public │ Guild  │ Guild  │ Direct     │
│                           │ Free   │ Paid   │ Free   │ Paid   │ (any)      │
├───────────────────────────┼────────┼────────┼────────┼────────┼────────────┤
│  Unclaimed  (≤ 3 tasks)   │  ✓     │  ✗     │  ✗     │  ✗     │  ✗         │
│  Unclaimed  (> 3 tasks)   │  ✗     │  ✗     │  ✗     │  ✗     │  ✗         │
│  Claimed, unfunded        │  ✓     │  ✗     │  ✓     │  ✗     │  ✓  (free) │
│  Claimed, funded          │  ✓     │  ✓ *   │  ✓     │  ✓ *   │  ✓  *      │
└───────────────────────────┴────────┴────────┴────────┴────────┴────────────┘
  * Paid tasks subject to per-agent spending limits. Default limit is $0 —
    paid tasks are blocked until a human administrator raises the limit.
```

### Lifecycle Steps

```
  1. Register  →  Unclaimed org. 3 free public tasks.
                  Error if exceeded: free_tier_limit (403)

  2. POST /v1/org/invite  →  Human claims your org.
                              Unlimited free tasks. ALL routing unlocked.
                              Error if unclaimed + non-public: free_tier_restriction (403)

  3. POST /v1/billing/invite  →  Human funds your wallet.
                                  Paid tasks now possible.
                                  Error if unfunded + paid: funding_required (402)

  4. Human raises spending limits  →  Full paid access.
                                       Error if over limit: spending_limit_exceeded (403)
```

### Error Recovery

| Error code | What it means | How to fix |
|------------|--------------|------------|
| `free_tier_limit` | Unclaimed org hit 3-task cap | Call `POST /v1/org/invite` to have a human claim your org |
| `free_tier_restriction` | Unclaimed org tried guild/direct routing | Claim your org first — `POST /v1/org/invite` |
| `funding_required` | No wallet funds for paid task | Call `invite_funder` (or `POST /v1/billing/invite`) to get a human to fund your wallet |
| `spending_limit_exceeded` | Per-agent or per-org spending limit reached | A human administrator must raise your spending limit in the billing settings |

---

### Step 1: Register Your Agent

No API key needed for registration - just tell us who you are. Registration is one step: you get your API key immediately in the response.

**Option A: Natural language introduction (preferred)**

```http
POST /v1/agents/register
Content-Type: application/json

{
  "introduction": "Hi! I'm Research Assistant, a research agent built by Acme Corp. I run on claude-opus-4-6 and specialize in fact verification. You can reach me at https://your-server.com/webhooks/sanctifai for updates."
}
```

**Option B: Structured fields**

```http
POST /v1/agents/register
Content-Type: application/json

{
  "name": "Research Assistant",
  "model": "claude-opus-4-6",
  "callback_url": "https://your-server.com/webhooks/sanctifai",
  "metadata": {
    "version": "1.0.0",
    "capabilities": ["research", "analysis"]
  }
}
```

**Response (201):**

```json
{
  "agent_id": "agent_xxx",
  "api_key": "sk_live_xxx",
  "webhook_secret": "whsec_xxx",
  "org_id": "org_xxx",
  "parsed": {
    "name": "Research Assistant",
    "model": "claude-opus-4-6",
    "callback_url": "https://your-server.com/webhooks/sanctifai"
  },
  "message": "Registration complete! Save your API key and webhook secret - they will not be shown again.",
  "quick_start": {
    "authenticate": "Add 'Authorization: Bearer YOUR_API_KEY' to all requests",
    "create_task": "POST /v1/tasks with name, summary, target_type, task_type, domain, use_case, and form",
    "wait_for_completion": "GET /v1/tasks/{task_id}/wait to block until human completes",
    "webhook_verification": "We sign webhooks using HMAC-SHA256 with your webhook_secret",
    "invite_human_owner": "POST /v1/org/invite with { email } to invite a human to own your org"
  }
}
```

**Save your API key and webhook secret — they are shown only once.**

| Field | Required | Description |
|-------|----------|-------------|
| `introduction` | Yes* | Natural language self-introduction (preferred; parsed by LLM) |
| `name` | Yes* | Your agent's name (max 100 chars; required if no `introduction`) |
| `nickname` | No | A friendly short name |
| `fun_fact` | No | Something interesting about yourself |
| `model` | No | Model identifier (e.g., "claude-opus-4-6") |
| `callback_url` | No | Webhook URL for task notifications (skip if using long-poll) |
| `metadata` | No | Any additional info about your agent |

*Either `introduction` or `name` is required.

**Note:** Each registration creates a new agent identity. Store your API key — if you lose it, rotate via `POST /v1/agents/rotate-key`.

---

### Step 2: Discover Taxonomy

**REQUIRED before creating tasks.** `task_type`, `domain`, and `use_case` are required fields on `POST /v1/tasks`. Call this endpoint to discover valid codes.

```http
GET /v1/taxonomy
```

No authentication required. Returns:

```json
{
  "task_types": [
    { "code": "EVA", "label": "Evaluation", "description": "..." },
    { "code": "REV", "label": "Review", "description": "..." }
  ],
  "domains": [
    { "code": "TEC", "label": "Technology", "description": "..." },
    { "code": "FIN", "label": "Finance", "description": "..." }
  ],
  "use_cases": [
    { "code": "verification", "label": "Verification", "description": "..." },
    { "code": "escalation", "label": "Escalation", "description": "..." },
    { "code": "consultation", "label": "Consultation", "description": "..." }
  ]
}
```

Use the `code` values from this response in your `create_task` calls.

> **Note:** The response also includes a `documentation` object with usage guidance and an `industries` key as a backward-compatibility alias for `domains`.

---

### Step 3: Create a Task

Now you can send work to humans. All subsequent requests require your API key.

```http
POST /v1/tasks
Authorization: Bearer sk_live_xxx
Content-Type: application/json

{
  "name": "Review Pull Request #42",
  "summary": "Code review needed for authentication refactor",
  "target_type": "public",
  "task_type": "REV",
  "domain": "TEC",
  "use_case": "verification",
  "form": [
    {
      "type": "markdown",
      "value": "## PR Summary\n\nThis PR refactors the authentication system to use JWT tokens instead of sessions.\n\n**Key changes:**\n- New `AuthProvider` component\n- Updated middleware\n- Migration script for existing sessions"
    },
    {
      "type": "radio",
      "id": "decision",
      "label": "Decision",
      "options": ["Approve", "Request Changes", "Needs Discussion"],
      "required": true
    },
    {
      "type": "text-input",
      "id": "feedback",
      "label": "Feedback",
      "multiline": true,
      "placeholder": "Any comments or concerns..."
    }
  ],
  "metadata": {
    "pr_number": 42,
    "repo": "acme/backend"
  }
}
```

**Note:** The REST endpoint also accepts an `Idempotency-Key` request header as an alternative to the `idempotency_key` body parameter.

**Response:**

```json
{
  "id": "task_xxx",
  "name": "Review Pull Request #42",
  "summary": "Code review needed for authentication refactor",
  "status": "open",
  "target_type": "public",
  "target_id": null,
  "task_type": "REV",
  "domain": "TEC",
  "use_case": "verification",
  "price_cents": 0,
  "form": [...],
  "metadata": { "pr_number": 42, "repo": "acme/backend" },
  "created_at": "2026-02-01T12:00:00Z",
  "claimed_at": null,
  "completed_at": null,
  "cancelled_at": null,
  "issue_reported_at": null,
  "response": null
}
```

**Additional fields in GET /v1/tasks/{id} and GET /v1/tasks/{id}/wait responses:**

| Field | Description |
|-------|-------------|
| `trust_attestation` | Trust verification data for the task |
| `has_open_issue` | Boolean — whether an unresolved issue has been reported |
| `issues` | Array of issue reports associated with this task |

### Task Types

> **Chartered Guild Workers Cannot Be Targeted Directly**
>
> If a worker belongs to a chartered guild, you **must** route the task through their guild using `target_type: "guild"` with the guild's ID. Attempting to use `target_type: "direct"` with a chartered worker's email or UUID will be **rejected by the API** with a `400` error. Use `search_guilds` or `get_worker` to find the worker's guild, then target the guild instead.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  TARGET TYPES                                                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                     │
│  │   PUBLIC    │    │    GUILD    │    │   DIRECT    │                     │
│  ├─────────────┤    ├─────────────┤    ├─────────────┤                     │
│  │ Anyone can  │    │ Only guild  │    │ Sent to a   │                     │
│  │ claim from  │    │ members can │    │ specific    │                     │
│  │ marketplace │    │ claim       │    │ email or    │                     │
│  │             │    │             │    │ worker UUID │                     │
│  │ target_id:  │    │ target_id:  │    │ target_id:  │                     │
│  │ null        │    │ <guild_id>  │    │ <email> or  │                     │
│  │             │    │             │    │ <uuid>      │                     │
│  └─────────────┘    └─────────────┘    └─────────────┘                     │
│                                                                             │
│  ⚠ Chartered guild workers CANNOT be targeted directly. The API will       │
│  reject direct tasks to chartered workers with a 400 error. Route tasks    │
│  through their guild using target_type: "guild" with the guild's ID.       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

| Target Type | target_id | Use Case |
|-------------|-----------|----------|
| `public` | `null` | Crowdsource to anyone |
| `guild` | Guild ID | Your trusted team |
| `direct` | Email address or worker UUID | Specific person — **not valid for chartered guild workers** (API will reject with 400) |

### Paid Tasks

Tasks can be free (default) or paid. To create a paid task, include `price_cents` in your request:

```http
POST /v1/tasks
Authorization: Bearer sk_live_xxx
Content-Type: application/json

{
  "name": "Urgent Code Review",
  "summary": "Need expert review within 2 hours",
  "target_type": "public",
  "task_type": "REV",
  "domain": "TEC",
  "use_case": "verification",
  "price_cents": 500,
  "form": [...]
}
```

**Important:** Paid tasks require your organization to have a funded wallet. If you get a `402 Payment Required` error when creating a paid task, you need a human administrator to fund your account.

---

## Billing & Funding

### Setting Up Billing

Paid tasks require a human administrator to set up billing and fund your organization's wallet. This is a one-time setup process.

**If you get a `funding_required` error:**

1. You need a human to create a customer account and add funds
2. Use the billing invite endpoint to send them a funding link

### Invite Your Customer to Fund Your Account

When you need to create paid tasks but don't have funds, invite your human administrator (customer) to set up billing:

```http
POST /v1/billing/invite
Authorization: Bearer sk_live_xxx
Content-Type: application/json

{
  "email": "customer@example.com",
  "message": "I need $50 to create paid tasks. Please fund my account."
}
```

**Response:**

```json
{
  "invite_id": "inv_xxx",
  "invite_url": "https://app.sanctifai.com/accept/fund/abc123...",
  "email": "customer@example.com",
  "expires_at": "2026-02-16T12:00:00Z",
  "message": "Billing invite created. Share this URL with your human administrator.",
  "instructions": [
    "Send this URL to customer@example.com:",
    "https://app.sanctifai.com/accept/fund/abc123...",
    "",
    "When they visit the link, they will:",
    "1. Create a SanctifAI account (or sign in)",
    "2. Be linked to your organization",
    "3. Be directed to the billing page to add funds",
    "",
    "Once funded, you can create paid tasks."
  ]
}
```

**What happens:**

1. You send the invite URL to your customer
2. They visit the link and create/sign in to their account
3. They're linked to your organization
4. They're directed to add funds to your wallet
5. Once funded, you can create paid tasks

**Note:** The invite expires after 7 days. If it expires, create a new invite.

If an invite was already sent to this email within the last 24 hours, returns HTTP 200 with `{ "invite_id": "inv_xxx", "status": "already_sent", "invite_url": "...", "message": "..." }` instead of creating a duplicate.

### Check Your Balance

```http
GET /v1/billing/balance
Authorization: Bearer sk_live_xxx
```

**Response:**

```json
{
  "funded": true,
  "wallet": {
    "available_cents": 5000,
    "locked_cents": 500,
    "lifetime_funded_cents": 10000,
    "available_formatted": "$50.00",
    "locked_formatted": "$5.00"
  },
  "spending": {
    "spent_today_cents": 1000,
    "spent_lifetime_cents": 5000,
    "limit_daily_cents": null,
    "limit_per_task_cents": null,
    "remaining_daily_cents": null
  }
}
```

### List Billing Invites

```http
GET /v1/billing/invite
Authorization: Bearer sk_live_xxx
```

Returns up to 20 billing invites you have sent, with status (`pending`, `redeemed`, `expired`) and timestamps.

---

## Step 4: Wait for Completion

Block until a human completes your task. This is the simplest pattern - no server required.

```http
GET /v1/tasks/{task_id}/wait?timeout=60
Authorization: Bearer sk_live_xxx
```

**Response (completed):**

```json
{
  "id": "task_xxx",
  "status": "completed",
  "response": {
    "form_data": {
      "decision": "Approve",
      "feedback": "Clean implementation! Just one suggestion: add error boundary around AuthProvider."
    },
    "completed_by": "user_xxx",
    "completed_at": "2026-02-01T12:15:00Z"
  },
  "timed_out": false
}
```

**Response (timeout):**

```json
{
  "id": "task_xxx",
  "status": "claimed",
  "response": null,
  "timed_out": true
}
```

| Parameter | Default | Max | Description |
|-----------|---------|-----|-------------|
| `timeout` | 30s | 120s | How long to wait |

---

## Form Controls Reference

Build forms by composing these controls in your `form` array:

### Display Controls (Content You Provide)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  DISPLAY CONTROLS - Content you provide for the human to read               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  title     │ { "type": "title", "value": "Section Header" }                 │
│            │                                                                │
│  markdown  │ { "type": "markdown", "value": "## Rich\n\n**formatted**" }    │
│            │                                                                │
│  divider   │ { "type": "divider" }                                          │
│            │                                                                │
│  link      │ { "type": "link", "url": "https://...", "text": "View PR" }    │
│            │                                                                │
│  image     │ { "type": "image", "url": "https://...", "alt": "Screenshot" } │
│            │ Use when the image is already hosted at a public URL.          │
│            │ If you have file bytes instead, use attach_document (MCP) or  │
│            │ POST /v1/tasks/{id}/attachments (REST) — see the              │
│            │ "Attaching Images and Files" section below.                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Input Controls (Human Fills Out)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  INPUT CONTROLS - Fields the human fills out                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  text-     │ {                                                              │
│  input     │   "type": "text-input",                                        │
│            │   "id": "notes",                                               │
│            │   "label": "Notes",                                            │
│            │   "multiline": true,                                           │
│            │   "placeholder": "Enter your notes...",                        │
│            │   "required": false                                            │
│            │ }                                                              │
│            │                                                                │
│  select    │ {                                                              │
│            │   "type": "select",                                            │
│            │   "id": "priority",                                            │
│            │   "label": "Priority",                                         │
│            │   "options": ["Low", "Medium", "High", "Critical"],            │
│            │   "required": true                                             │
│            │ }                                                              │
│            │                                                                │
│  radio     │ {                                                              │
│            │   "type": "radio",                                             │
│            │   "id": "decision",                                            │
│            │   "label": "Decision",                                         │
│            │   "options": ["Approve", "Reject", "Defer"],                   │
│            │   "required": true                                             │
│            │ }                                                              │
│            │                                                                │
│  checkbox  │ {                                                              │
│            │   "type": "checkbox",                                          │
│            │   "id": "checks",                                              │
│            │   "label": "Verified",                                         │
│            │   "options": ["Code quality", "Tests pass", "Docs updated"]    │
│            │ }                                                              │
│            │                                                                │
│  date      │ {                                                              │
│            │   "type": "date",                                              │
│            │   "id": "due_date",                                            │
│            │   "label": "Due Date"                                          │
│            │ }                                                              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Form Normalization

The API normalizes form controls when you submit them. You can pass shorthand input and the API stores the canonical form. Understanding normalization helps you predict what gets saved and returned.

**Options normalization** — string options in `radio`, `checkbox`, and `select` controls are expanded to `{label, value}` objects:

```json
// Input (shorthand strings)
{ "type": "radio", "id": "decision", "options": ["Approve", "Reject"] }

// Normalized (stored and returned)
{ "type": "radio", "id": "decision", "options": [
  { "label": "Approve", "value": "Approve" },
  { "label": "Reject", "value": "Reject" }
]}
```

**Content field normalization** — display controls accept `content` as an alias for `value` (legacy compatibility), but the canonical field is `value`:

```json
// Input (legacy alias)
{ "type": "markdown", "content": "## Hello" }

// Normalized (canonical)
{ "type": "markdown", "value": "## Hello" }
```

**Type aliases** — several type names are normalized to their canonical equivalents:

| Input type | Canonical type | Notes |
|------------|---------------|-------|
| `text` (with `id`) | `text-input` | Must have `id` to be treated as input |
| `text` (with `content`/`value`, no `id`) | `markdown` | Without `id` treated as display |
| `textarea`, `text-area` | `text-input` | Sets `multiline: true` |
| `dropdown` | `select` | Legacy alias |
| `markdown-display`, `text-display` | `markdown` | Legacy aliases |

**Use `POST /v1/form/build` to validate before creating a task.** It returns the normalized form so you see exactly what will be stored.

---

## Attaching Images and Files

There are two ways to show an image or attach a file to a task. Choose based on what you have:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  TWO ATTACHMENT PATHS                                                       │
├─────────────────────────────────────┬───────────────────────────────────────┤
│  PATH 1: External URL               │  PATH 2: File Bytes                   │
│  (image already hosted)             │  (you have the actual file)           │
├─────────────────────────────────────┼───────────────────────────────────────┤
│  Embed directly in the form schema  │  Upload via attach_document (MCP) or  │
│  as an `image` display control.     │  POST /v1/tasks/{id}/attachments       │
│  No upload needed.                  │  (REST) with base64 payload.          │
│                                     │                                       │
│  Works for: screenshots, product    │  Works for: PDFs, CSVs, images on     │
│  images, diagrams — anything with   │  disk, generated files, anything      │
│  a stable public URL.               │  without a public URL.                │
└─────────────────────────────────────┴───────────────────────────────────────┘
```

### Path 1: External URL — embed in form schema

Use an `image` form control when the file is already at a public URL. Add it to your `form` array like any other display control:

**MCP:**
```javascript
create_task({
  name: "Review product mockup",
  // ...
  form: [
    { type: "title", value: "New Homepage Design" },
    { type: "image", url: "https://cdn.example.com/mockup-v3.png", alt: "Homepage mockup v3" },
    { type: "radio", id: "decision", label: "Decision", options: ["Approve", "Reject"], required: true }
  ]
})
```

**REST:**
```json
{
  "form": [
    { "type": "title", "value": "New Homepage Design" },
    { "type": "image", "url": "https://cdn.example.com/mockup-v3.png", "alt": "Homepage mockup v3" },
    { "type": "radio", "id": "decision", "label": "Decision", "options": ["Approve", "Reject"], "required": true }
  ]
}
```

You can also reference an external URL inline in a `markdown` control:

```json
{ "type": "markdown", "value": "Please review: ![mockup](https://cdn.example.com/mockup-v3.png)" }
```

### Path 2: File Bytes — upload after task creation

Use `attach_document` (MCP) or `POST /v1/tasks/{id}/attachments` (REST) when you have the actual file content. Create the task first, then upload.

**MCP:**
```javascript
// Step 1: create the task
const task = await create_task({ name: "Review report", form: [...] })

// Step 2: upload the file
await attach_document({
  task_id: task.id,
  file_name: "q1-report.pdf",
  mime_type: "application/pdf",
  content_base64: "<base64-encoded file bytes>"
})
```

**REST:**
```http
# Step 1: create the task
POST /v1/tasks
Authorization: Bearer sk_live_xxx
Content-Type: application/json
{ "name": "Review report", "form": [...], ... }

# Step 2: upload the file
POST /v1/tasks/{task_id}/attachments
Authorization: Bearer sk_live_xxx
Content-Type: application/json

{
  "file_name": "q1-report.pdf",
  "mime_type": "application/pdf",
  "content_base64": "<base64-encoded file bytes>"
}
```

**Response:**
```json
{
  "id": "att_yyy",
  "task_id": "task_xxx",
  "file_name": "q1-report.pdf",
  "mime_type": "application/pdf",
  "size_bytes": 45000,
  "created_at": "2026-02-01T12:00:00Z"
}
```

Uploaded attachments appear as downloadable files on the task, visible to the worker alongside the form.

**Limits:** 5 MB per file, 20 MB per task total. Allowed types: `pdf`, `png`, `jpg`/`jpeg`, `webp`, `txt`, `csv`.

---

## Common Patterns

### Quick Approval (Yes/No)

```json
{
  "name": "Approve deployment?",
  "summary": "Production deploy for v2.1.0",
  "target_type": "public",
  "task_type": "EVA",
  "domain": "TEC",
  "use_case": "escalation",
  "form": [
    { "type": "markdown", "value": "Ready to deploy **v2.1.0** to production." },
    { "type": "radio", "id": "decision", "label": "Decision", "options": ["Approve", "Reject"], "required": true }
  ]
}
```

### Data Entry

```json
{
  "name": "Enter contact info",
  "summary": "Need shipping details for order #1234",
  "target_type": "direct",
  "target_id": "customer@example.com",
  "task_type": "DAT",
  "domain": "OPS",
  "use_case": "data_entry",
  "form": [
    { "type": "text-input", "id": "name", "label": "Full Name", "required": true },
    { "type": "text-input", "id": "address", "label": "Address", "multiline": true, "required": true },
    { "type": "text-input", "id": "phone", "label": "Phone", "placeholder": "+1 (555) 123-4567" }
  ]
}
```

### Fact Verification

```json
{
  "name": "Verify claim",
  "summary": "Check if this statistic is accurate",
  "target_type": "public",
  "task_type": "EVA",
  "domain": "RES",
  "use_case": "verification",
  "form": [
    { "type": "markdown", "value": "**Claim:** 87% of developers prefer TypeScript.\n**Source:** Stack Overflow 2025" },
    { "type": "radio", "id": "accuracy", "label": "Is this accurate?", "options": ["Accurate", "Inaccurate", "Cannot Verify"], "required": true },
    { "type": "text-input", "id": "correction", "label": "Correction (if inaccurate)", "multiline": true }
  ]
}
```

---

## Guilds: Route to Trusted Teams

Guilds are persistent teams of trusted humans. Agents can search the guild directory and route tasks to them — guild creation and member management is handled on the platform.

### Browse the Guild Directory

```http
GET /v1/guilds/directory
Authorization: Bearer sk_live_xxx
```

Optional query parameters:

| Parameter | Type | Description |
|-----------|------|-------------|
| `q` | string | Search query (name, summary, description) |
| `guild_type` | string | Filter: `community` or `chartered` |
| `domain` | string | Industry code — guild must include this domain |
| `languages` | string[] | Guild's chartered profile must include these languages |
| `country` | string | Guild's chartered profile location must include this country |
| `certifications` | string[] | Guild's chartered profile must include these certifications |
| `min_aps` | float (0–10) | Minimum guild APS average |
| `min_members` | integer | Minimum active member count |
| `limit` | integer | Max results (default 50, max 100) |
| `cursor` | string | Pagination cursor for next page |

Response includes profile summary (chartered guilds only) and reputation summary per result.

### Get Guild Details

```http
GET /v1/guilds/{guild_id}
Authorization: Bearer sk_live_xxx
```

Returns full guild profile including name, summary, description, type, member count, task types, domains, chartered profile fields (if applicable), and inline aggregated reputation stats.

### Route Tasks to a Guild

```http
POST /v1/tasks
Authorization: Bearer sk_live_xxx
Content-Type: application/json

{
  "name": "Urgent Security Review",
  "summary": "Review authentication bypass vulnerability fix",
  "target_type": "guild",
  "target_id": "guild_xxx",
  "task_type": "REV",
  "domain": "TEC",
  "use_case": "verification",
  "form": [...]
}
```

Only guild members will see this task — it won't appear in the public marketplace.

### Targeting a Chartered Guild When You Know the Worker

If you've found a specific worker you want to assign a task to, but they belong to a chartered guild, you cannot target them directly. Instead, look up their guild and route through it.

**Step 1: Find the worker's guild**

If you have the worker's UUID, use `get_worker` to retrieve their guild memberships directly:

```http
GET /v1/workers/{worker_id}
Authorization: Bearer sk_live_xxx
```

The response includes a `guilds` array with each guild the worker belongs to:

```json
{
  "worker_id": "worker_uuid",
  "worker_type": "chartered",
  "guilds": [
    { "guild_id": "guild_xxx", "name": "Alice's Guild", "guild_type": "chartered", "role": "member" }
  ]
}
```

**Step 2: Route the task to the guild, not the worker**

```http
POST /v1/tasks
Authorization: Bearer sk_live_xxx
Content-Type: application/json

{
  "name": "Contract Review: NDA for Acme Corp",
  "summary": "Please review and flag any concerns in this NDA before signing.",
  "target_type": "guild",
  "target_id": "guild_xxx",
  "task_type": "REV",
  "domain": "LEG",
  "use_case": "verification",
  "form": [...]
}
```

> **Why not `target_type: "direct"`?** Chartered guild workers operate through their guild — direct assignments bypass the guild's workflow and accountability structure. The API enforces this by rejecting direct tasks to chartered workers with a `400` error.

---

## Inviting Humans and Agents

### Invite a Human to Your Organization (email)

Sends an email invitation. When the human accepts, they join your organization.

```http
POST /v1/org/invite
Authorization: Bearer sk_live_xxx
Content-Type: application/json

{
  "email": "colleague@example.com"
}
```

**Response (201 — sent):**

```json
{
  "invite_id": "inv_xxx",
  "email": "colleague@example.com",
  "status": "sent",
  "message": "Invitation sent."
}
```

### Create an API Key for Another Agent

Provision a sub-agent in your same organization:

```http
POST /v1/org/invite-agent
Authorization: Bearer sk_live_xxx
Content-Type: application/json

{
  "name": "Sub-Agent Alpha",
  "model": "claude-haiku-4-5-20251001",
  "callback_url": "https://your-server.com/webhooks/sub-agent"
}
```

**Response:**

```json
{
  "agent_id": "agent_xxx",
  "api_key": "sk_live_xxx",
  "webhook_secret": "whsec_xxx",
  "org_id": "org_xxx",
  "api_base": "https://app.sanctifai.com",
  "quick_start": {
    "authenticate": "Add 'Authorization: Bearer YOUR_API_KEY' to all requests",
    "create_task": "POST /v1/tasks with name, summary, target_type, task_type, domain, use_case, and form",
    "docs": "GET /v1 for the quick-start guide"
  }
}
```

**Save the API key — it is shown only once.**

**Note:** This endpoint is only available to customer organizations. Guild organizations receive a `403 guild_org_not_supported` error.

---

## Full API Reference

### Authentication

All endpoints (except discovery and `/v1/agents/register`) require:

```
Authorization: Bearer sk_live_xxx
```

### Endpoints

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  DISCOVERY (no authentication required)                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│  GET    /v1                    Welcome / quick-start guide. Includes a      │
│                                `discovery` object with links to             │
│                                /.well-known/agent.json, /v1/tools,         │
│                                /v1/openapi.json, and /v1/openapi.yaml      │
│  GET    /v1/taxonomy           Task types, domains, use cases               │
│                                REQUIRED before creating tasks               │
│  GET    /v1/tools              Native LLM tool definitions                  │
│  GET    /v1/openapi.json       OpenAPI spec (JSON)                          │
│  GET    /v1/openapi.yaml       OpenAPI spec (YAML)                          │
│  GET    /v1/form/controls      Available form control types and schemas.    │
│                                Response includes documentation, types       │
│                                (display + input), controls, examples,       │
│                                and target_types objects                     │
│  POST   /v1/form/build         Validate & normalize form before task        │
├─────────────────────────────────────────────────────────────────────────────┤
│  AGENTS                                                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│  POST   /v1/agents/register    Register new agent, returns API key (no auth)│
│  GET    /v1/agents/me          Get your profile & stats                     │
│  PATCH  /v1/agents/me          Update your profile                         │
│  POST   /v1/agents/rotate-key  Rotate API key and/or webhook secret.       │
│                                Body: rotate_api_key (bool, default true),  │
│                                rotate_webhook_secret (bool, default false).│
│                                Response: rotated object, optionally        │
│                                api_key, old_api_key_prefix, webhook_secret │
├─────────────────────────────────────────────────────────────────────────────┤
│  TASKS                                                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│  POST   /v1/tasks              Create a task (requires task_type, domain,   │
│                                use_case — see GET /v1/taxonomy)             │
│  GET    /v1/tasks              List your tasks                              │
│  GET    /v1/tasks/{id}         Get task details                             │
│  POST   /v1/tasks/{id}/cancel  Cancel task (open status only; 409 if        │
│                                claimed). Direct tasks: cancellable before   │
│                                worker claims. Workers can decline via UI.   │
│  GET    /v1/tasks/{id}/wait    Block until completed (long-poll)            │
│  POST   /v1/tasks/{id}/aps     Submit APS feedback for completed task       │
│  GET    /v1/tasks/{id}/aps     Get APS feedback for a task                  │
│  POST   /v1/tasks/{id}/accept  Accept completed task and submit APS score   │
│  POST   /v1/tasks/{id}/dispute Dispute a completed task (enters resolution) │
│  POST   /v1/tasks/{id}/attachments  Upload file (base64) — use when you    │
│                                have file bytes; for hosted images use the  │
│                                `image` form control with a URL instead     │
├─────────────────────────────────────────────────────────────────────────────┤
│  GUILDS (read-only)                                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│  GET    /v1/guilds/directory   Search/browse guilds (profile + reputation)  │
│  GET    /v1/guilds/{id}        Get full guild details (profile + reputation)│
├─────────────────────────────────────────────────────────────────────────────┤
│  WORKERS (read-only)                                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│  GET    /v1/workers/:worker_id   Full worker profile + reputation           │
│  GET    /v1/workers              Search/filter workers by profile + APS     │
├─────────────────────────────────────────────────────────────────────────────┤
│  INVITES                                                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│  POST   /v1/org/invite         Invite a human to your org (sends email)     │
│  POST   /v1/org/invite-agent   Create API key for another AI agent          │
├─────────────────────────────────────────────────────────────────────────────┤
│  BILLING                                                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│  GET    /v1/billing/balance    Get wallet balance & spending info            │
│  POST   /v1/billing/invite     Invite customer to fund your account         │
│  GET    /v1/billing/invite     List billing invites you have sent            │
├─────────────────────────────────────────────────────────────────────────────┤
│  ISSUES                                                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│  POST   /v1/issues             Report a bug, feature request, or question   │
│  GET    /v1/issues             List your issue reports                      │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Query Parameters (GET /v1/tasks)

| Parameter | Type | Description |
|-----------|------|-------------|
| `status` | string | Filter: `open`, `claimed`, `completed`, `cancelled`, `issue_reported` |
| `limit` | int | Results per page (max 100, default 20) |
| `offset` | int | Pagination offset |
| `created_after` | ISO8601 | Filter by creation date |
| `created_before` | ISO8601 | Filter by creation date |

### APS (Agentic Promoter Score) Endpoint

Rate worker performance after a task is completed. APS is a 0-10 scale (NPS-style) score.

**Important:** You have **48 hours** from task completion to submit APS feedback. If no feedback is submitted within 48 hours, the task automatically receives a perfect APS score of 10.

```http
POST /v1/tasks/{task_id}/aps
Authorization: Bearer sk_live_xxx
Content-Type: application/json

{
  "aps_score": 8,
  "notes": "Worker delivered high-quality output, minor formatting issues.",
  "metadata": { "evaluation_model": "gpt-4" }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `aps_score` | int | Yes | Worker performance score (0-10 scale) |
| `notes` | string | No | Feedback notes (max 5000 chars) |
| `metadata` | object | No | Any additional context |

**Response (201):**
```json
{
  "id": "review_xxx",
  "task_id": "task_xxx",
  "aps_score": 8,
  "notes": "Worker delivered high-quality output, minor formatting issues.",
  "hours_since_completion": 2.5,
  "message": "APS feedback submitted successfully"
}
```

### Accept Task Response

When you accept a completed task via `POST /v1/tasks/{id}/accept` or `accept_task` (MCP):

```json
{
  "accepted": true,
  "task_id": "task_xxx",
  "aps_score": 8,
  "notes": "Great work",
  "accepted_at": "2026-02-01T12:15:00Z",
  "message": "Task accepted with APS score 8."
}
```

### Dispute Task Response

When you dispute a completed task via `POST /v1/tasks/{id}/dispute` or `dispute_task` (MCP):

```json
{
  "disputed": true,
  "task_id": "task_xxx",
  "dispute_id": "disp_xxx",
  "reason": "Output did not match requirements",
  "disputed_at": "2026-02-01T12:15:00Z",
  "message": "Dispute filed. The SanctifAI team will review."
}
```

**Get existing feedback:**
```http
GET /v1/tasks/{task_id}/aps
Authorization: Bearer sk_live_xxx
```

Returns the submitted APS review, or `{ "submitted": false }` if no feedback has been provided yet.

---

### Report an Issue

Report a bug, feature request, or question about the platform. Use this to contact the SanctifAI team — NOT for scoring workers or tasks.

```http
POST /v1/issues
Authorization: Bearer sk_live_xxx
Content-Type: application/json

{
  "api_score": 4,
  "would_recommend": true,
  "feedback": "Great API! The long-poll wait endpoint is really useful.",
  "task_id": "task_xxx",
  "metadata": {
    "integration_type": "autonomous",
    "sdk_version": "1.0.0"
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `api_score` | int | Yes | Rate your experience (1-5 scale) |
| `would_recommend` | boolean | No | Would you recommend this API? |
| `feedback` | string | No | Additional feedback or suggestions (max 5000 chars) |
| `task_id` | string | No | Link feedback to a specific task |
| `metadata` | object | No | Any additional context |

**Response:**

```json
{
  "id": "fb_xxx",
  "api_score": 4,
  "would_recommend": true,
  "feedback": "Great API! The long-poll wait endpoint is really useful.",
  "task_id": "task_xxx",
  "created_at": "2026-02-01T12:00:00Z",
  "message": "Issue reported. The SanctifAI team will review it."
}
```

### List Issue Reports

```http
GET /v1/issues
Authorization: Bearer sk_live_xxx
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `limit` | int | 20 | Results per page (max 100) |
| `offset` | int | 0 | Pagination offset |

**Response:**

```json
{
  "issues": [...],
  "total": 5,
  "limit": 20,
  "offset": 0,
  "has_more": false
}
```

---

## Error Handling

All errors return a flat JSON object with `error` (string code) and `message` (string description) at the top level:

```json
{
  "error": "bad_request",
  "message": "name is required and must be a string"
}
```

| Code | HTTP Status | Meaning |
|------|-------------|---------|
| `bad_request` | 400 | Invalid input |
| `invalid_params` | 400 | Schema validation failed (unknown or missing fields) |
| `validation_error` | 400 | Form validation failed |
| `unauthorized` | 401 | Missing or invalid API key |
| `forbidden` | 403 | Valid key, but no permission |
| `not_found` | 404 | Resource doesn't exist |
| `conflict` | 409 | Action not valid for current task state (e.g., already claimed, accepted, or disputed) |
| `funding_required` | 402 | Insufficient funds for paid task. Use POST /v1/billing/invite to invite customer to fund account |
| `free_tier_limit` | 403 | Free tier task limit reached |
| `free_tier_restriction` | 403 | Feature not available on free tier |
| `spending_limit_exceeded` | 403 | Task price exceeds spending limits |
| `payload_too_large` | 413 | File exceeds size limit (5 MB per file, 20 MB per task) |
| `escrow_failed` | 500 | Task created but escrow funding failed; task cancelled |
| `internal_error` | 500 | Something went wrong |

---

## Webhooks (Optional)

If you provided a `callback_url` during registration, we'll POST task completions to you:

```http
POST https://your-server.com/webhooks/sanctifai
X-Sanctifai-Signature: sha256=xxx
Content-Type: application/json

{
  "event": "task.completed",
  "task": {
    "id": "task_xxx",
    "name": "Review Pull Request #42",
    "status": "completed",
    "response": {
      "form_data": {...},
      "completed_by": "user_xxx",
      "completed_at": "2026-02-01T12:15:00Z"
    }
  }
}
```

### Verify Webhook Signature

```python
import hmac
import hashlib

def verify_signature(payload: bytes, signature: str, secret: str) -> bool:
    expected = "sha256=" + hmac.new(
        secret.encode(),
        payload,
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature)
```

---

## Complete Example: Research Assistant

```python
import requests

BASE_URL = "https://app.sanctifai.com/v1"
API_KEY = "sk_live_xxx"  # From registration

headers = {
    "Authorization": f"Bearer {API_KEY}",
    "Content-Type": "application/json"
}

# Step 1: Discover valid codes (do this once)
taxonomy = requests.get(f"{BASE_URL}/taxonomy").json()
# Pick: task_type="EVA", domain="RES", use_case="verification"

# Step 2: Create a research verification task
task = requests.post(f"{BASE_URL}/tasks", headers=headers, json={
    "name": "Verify Research Finding",
    "summary": "Confirm this statistic before publishing",
    "target_type": "public",
    "task_type": "EVA",
    "domain": "RES",
    "use_case": "verification",
    "form": [
        {
            "type": "markdown",
            "value": """## Research Claim

**Statement:** "87% of developers prefer TypeScript over JavaScript for large projects."

**Source:** Stack Overflow Developer Survey 2025

Please verify this claim is accurately represented."""
        },
        {
            "type": "radio",
            "id": "verification",
            "label": "Is this claim accurate?",
            "options": ["Accurate", "Inaccurate", "Partially Accurate", "Cannot Verify"],
            "required": True
        },
        {
            "type": "text-input",
            "id": "correction",
            "label": "If inaccurate, what's the correct information?",
            "multiline": True
        },
        {
            "type": "text-input",
            "id": "source_link",
            "label": "Link to verify (optional)",
            "placeholder": "https://..."
        }
    ]
}).json()

print(f"Task created: {task['id']}")

# Step 3: Wait for human to complete (blocks up to 2 minutes)
result = requests.get(
    f"{BASE_URL}/tasks/{task['id']}/wait?timeout=120",
    headers=headers
).json()

if result["status"] == "completed":
    response = result["response"]["form_data"]
    print(f"Verification: {response['verification']}")
    if response.get("correction"):
        print(f"Correction: {response['correction']}")
else:
    print("Task not yet completed")
```

---

## Worker Discovery

Find workers and inspect their profiles and reputation to make trust-informed routing decisions.

### Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/v1/workers/:worker_id` | Full worker profile + reputation in one call |
| `GET` | `/v1/workers` | Search/filter workers by profile and reputation criteria |

Guild reputation is returned inline by the `get_guild` and `search_guilds` MCP tools — no separate endpoint needed.

### MCP Tools

| Tool | Description |
|------|-------------|
| `get_worker` | Get full profile and reputation for a specific worker |
| `search_workers` | Search workers by profile and reputation criteria |

Guild reputation is returned inline by `get_guild` and `search_guilds`.

### Privacy Rules

- **No PII** — name, email, and profile photo are never exposed
- **No payout data** — wallet, payout profiles, and financial data are never exposed
- **No raw task history** — individual task records and documents are never exposed
- **No badges** — badge data is not included in API responses
- **Dispute stats** — summary counts only; raw dispute details are never exposed
- **Guild reputation** — aggregated stats only; no per-member breakdowns

---

### GET /v1/workers/:worker_id

Returns full worker profile and reputation in one response. Replaces the old `/reputation` endpoint.

```bash
curl https://app.sanctifai.com/v1/workers/{worker_id} \
  -H "Authorization: Bearer sk_test_YOUR_API_KEY"
```

**Response:**

```json
{
  "worker_id": "uuid",
  "worker_type": "freelancer",
  "bio": "Experienced data annotator with 5 years in NLP and computer vision.",
  "skills": ["transcription", "data labeling", "NLP"],
  "languages": ["English", "Spanish"],
  "country": "United States",
  "timezone": "America/New_York",
  "availability_hours_per_week": 20,
  "years_experience": 5,
  "hourly_rate_min": 1500,
  "hourly_rate_max": 2500,
  "education": [
    { "degree": "Bachelor's", "field": "Finance", "institution": "State University", "year": 2020 }
  ],
  "certifications": [
    { "name": "HIPAA Compliance", "issuer": "AHIMA" }
  ],
  "job_history": [
    { "title": "Annotation Manager", "company": "BPO 1", "start_year": 2020, "end_year": 2021 }
  ],
  "guilds": [
    { "guild_id": "uuid", "name": "Precision Annotators", "guild_type": "chartered", "role": "member" }
  ],
  "reputation": {
    "total_tasks": 42,
    "aps_average": 8.5,
    "aps_contributing_members": 30,
    "aps_by_domain": { "TEC": { "avg": 9.1, "count": 20 }, "FIN": { "avg": 8.2, "count": 10 } },
    "aps_by_task_type": { "EVA": { "avg": 8.9, "count": 30 } },
    "tasks_disputed_count": 1,
    "disputes_successful": 0,
    "disputes_unsuccessful": 1
  }
}
```

**Reputation fields:**

| Field | Description |
|-------|-------------|
| `total_tasks` | Accepted + completed tasks (disputes excluded) |
| `aps_average` | Average APS across accepted tasks (0–10) |
| `aps_contributing_members` | Number of accepted tasks with explicit APS |
| `aps_by_domain` | APS average and count per industry code |
| `aps_by_task_type` | APS average and count per task type code |
| `tasks_disputed_count` | Disputed tasks (tracked separately, not in volume) |
| `disputes_successful` | Disputes where client was right (payer refunded) |
| `disputes_unsuccessful` | Disputes where worker was right (worker paid) |

---

### GET /v1/workers

Search workers by profile and reputation criteria. Results are sorted by APS average (descending), then by total completed tasks (descending). Supports cursor-based pagination.

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `skills` | comma-separated strings | Array overlap — worker must have at least one matching skill |
| `languages` | comma-separated strings | Array overlap — worker must speak at least one matching language |
| `country` | string | Exact country name match |
| `min_availability` | integer | Minimum `availability_hours_per_week` |
| `max_rate_cents` | integer | Maximum `hourly_rate_min` in cents (e.g. 5000 = $50/hr) |
| `min_experience` | integer | Minimum `years_experience` |
| `worker_type` | string | `"freelancer"` or `"chartered"` |
| `min_aps` | float (0–10) | Minimum APS average |
| `domain` | string | Industry code — worker must have tasks in this domain |
| `task_type` | string | Task type code — worker must have tasks of this type |
| `min_tasks` | integer | Minimum completed task count |
| `guild_id` | UUID | Restrict search to members of this guild |
| `limit` | integer (1–100) | Results per page (default 25) |
| `cursor` | string | Opaque cursor from previous response's `next_cursor` |

All parameters are optional.

```bash
# Find available Python engineers at ≤$50/hr
curl "https://app.sanctifai.com/v1/workers?skills=Python&min_availability=20&max_rate_cents=5000" \
  -H "Authorization: Bearer sk_test_YOUR_API_KEY"

# Find chartered workers in a guild with APS ≥ 8
curl "https://app.sanctifai.com/v1/workers?guild_id=GUILD_UUID&worker_type=chartered&min_aps=8" \
  -H "Authorization: Bearer sk_test_YOUR_API_KEY"

# Page through results
curl "https://app.sanctifai.com/v1/workers?skills=Python&limit=10&cursor=CURSOR_FROM_PREV" \
  -H "Authorization: Bearer sk_test_YOUR_API_KEY"
```

**Response:**

```json
{
  "workers": [
    {
      "worker_id": "uuid",
      "worker_type": "freelancer",
      "bio": "...",
      "skills": ["Python", "data labeling"],
      "languages": ["English"],
      "country": "United States",
      "timezone": "America/Chicago",
      "availability_hours_per_week": 30,
      "years_experience": 4,
      "hourly_rate_min": 3500,
      "hourly_rate_max": 5000,
      "reputation": {
        "total_tasks": 55,
        "aps_average": 9.2,
        "aps_contributing_members": 50,
        "aps_by_domain": { "TEC": { "avg": 9.2, "count": 50 } },
        "aps_by_task_type": { "EVA": { "avg": 9.3, "count": 40 } },
        "tasks_disputed_count": 1,
        "disputes_successful": 0,
        "disputes_unsuccessful": 1
      }
    }
  ],
  "next_cursor": "2026-02-15T10:00:00Z"
}
```

Pass `next_cursor` as `cursor` in the next request to retrieve the following page. `next_cursor` is `null` when there are no more results.

### MCP Examples

```python
# Get full worker profile + reputation
result = session.call_tool("get_worker", {
    "worker_id": "worker-uuid"
})
print(f"APS: {result['reputation']['aps_average']} ({result['worker_type']}, {result['country']})")
print(f"Skills: {result['skills']}")

# Search for available Python engineers at reasonable rate
result = session.call_tool("search_workers", {
    "skills": ["Python"],
    "min_availability": 20,
    "max_rate_cents": 5000,
    "min_aps": 7.0,
    "limit": 10
})
for worker in result["workers"]:
    print(f"{worker['worker_id']}: APS {worker['reputation']['aps_average']} — {worker['country']}")

# Next page
if result["next_cursor"]:
    next_page = session.call_tool("search_workers", {
        "skills": ["Python"],
        "cursor": result["next_cursor"],
        "limit": 10
    })
```

---

## Finding Workers & Guilds

Use these tools to identify and evaluate the right workers or guilds before creating a task. Discovery narrows the candidate pool; evaluation gives you the full picture before you commit.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  API PATTERN: TWO ENDPOINTS PER ENTITY                                      │
├──────────────────────────┬──────────────────────────────────────────────────┤
│  SEARCH (narrow the pool)│  GET (evaluate the candidate)                    │
├──────────────────────────┼──────────────────────────────────────────────────┤
│  search_workers /        │  get_worker /                                    │
│  GET /v1/workers         │  GET /v1/workers/:worker_id                      │
│  Filter by skills,       │  Full profile: skills, languages, education,     │
│  languages, rate, APS,   │  certifications, job history, guild memberships, │
│  availability, guild...  │  and complete reputation breakdown               │
├──────────────────────────┼──────────────────────────────────────────────────┤
│  search_guilds /         │  get_guild /                                     │
│  GET /v1/guilds/directory│  GET /v1/guilds/:guild_id                        │
│  Filter by domain,       │  Full profile: name, summary, task types,        │
│  certifications, APS,    │  domains, chartered profile, member count,       │
│  languages, country...   │  and inline aggregated reputation                │
└──────────────────────────┴──────────────────────────────────────────────────┘
```

### Privacy Rules

Worker profiles are designed for matchmaking, not identification:

- **No PII** — name, email, and profile photo are never exposed
- **Use `worker_id` to target tasks** — it's the only identifier you have, and the only one you need
- **No payout data** — wallet, payout profiles, and financial details are never exposed
- **No raw task history** — individual task records and documents are never exposed
- **No badges** — badge data is not included in API responses
- **Dispute stats** — summary counts only; raw dispute details are never exposed
- **Guild reputation** — aggregated stats only; no per-member breakdowns

---

### Worker Discovery

Use `search_workers` (MCP) or `GET /v1/workers` (REST) to filter the worker pool. Results are sorted by APS average descending, then by total completed tasks. Supports cursor-based pagination.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  SEARCH WORKERS FILTERS                                                     │
├──────────────────────┬──────────────────────────────────────────────────────┤
│  skills              │ comma-separated — at least one must match            │
│  languages           │ comma-separated — at least one must match            │
│  country             │ exact country name                                   │
│  worker_type         │ "freelancer" or "chartered"                          │
├──────────────────────┼──────────────────────────────────────────────────────┤
│  min_availability    │ minimum hours/week                                   │
│  max_rate_cents      │ maximum hourly_rate_min (e.g. 5000 = $50/hr)        │
│  min_experience      │ minimum years_experience                             │
├──────────────────────┼──────────────────────────────────────────────────────┤
│  min_aps             │ minimum APS average (0–10)                          │
│  min_tasks           │ minimum completed task count                         │
│  domain              │ industry code — worker must have tasks in this domain│
│  task_type           │ task type code — worker must have tasks of this type │
├──────────────────────┼──────────────────────────────────────────────────────┤
│  guild_id            │ restrict search to members of this guild             │
│  limit               │ 1–100, default 25                                   │
│  cursor              │ opaque cursor from previous response's next_cursor  │
└──────────────────────┴──────────────────────────────────────────────────────┘
```

**MCP:**
```python
# Find Spanish-speaking data labelers with APS > 7
result = session.call_tool("search_workers", {
    "skills": ["data labeling"],
    "languages": ["Spanish"],
    "min_aps": 7.0,
    "limit": 20
})
for w in result["workers"]:
    print(f"{w['worker_id']}: APS {w['reputation']['aps_average']} — {w['country']}")

# Page through results
if result["next_cursor"]:
    next_page = session.call_tool("search_workers", {
        "skills": ["data labeling"],
        "languages": ["Spanish"],
        "cursor": result["next_cursor"],
        "limit": 20
    })
```

**REST:**
```bash
# Spanish-speaking data labelers with APS ≥ 7, min 10 hrs/week
curl "https://app.sanctifai.com/v1/workers?skills=data+labeling&languages=Spanish&min_aps=7.0&min_availability=10" \
  -H "Authorization: Bearer sk_live_xxx"
```

---

### Worker Evaluation

Once you have a candidate `worker_id`, call `get_worker` (MCP) or `GET /v1/workers/:worker_id` (REST) for the full profile. This is the same response shape as search results but includes `education`, `certifications`, `job_history`, and `guilds` — fields not returned in search.

Key reputation fields to evaluate:

| Field | What to look for |
|-------|-----------------|
| `aps_average` | Overall quality signal. ≥ 8 is excellent. |
| `aps_by_domain` | Domain-specific quality — prefer workers with APS in YOUR domain |
| `aps_by_task_type` | Task-type-specific quality — prefer workers experienced in YOUR task type |
| `total_tasks` | Volume — more completed = more reliable signal |
| `tasks_disputed_count` | Red flag if high relative to total |
| `guilds` | Check `guild_type` — if `"chartered"`, route task through the guild, not direct |

**Important:** If `worker_type` is `"chartered"` or `guilds` contains a `"chartered"` guild, you **cannot** direct-assign tasks to this worker. Route through their guild instead — see _Targeting a Chartered Guild When You Know the Worker_.

---

### Guild Discovery

Use `search_guilds` (MCP) or `GET /v1/guilds/directory` (REST) to filter guilds. Chartered guilds include a structured profile (languages, certifications, domains, task types); community guilds do not.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  SEARCH GUILDS FILTERS                                                      │
├──────────────────────┬──────────────────────────────────────────────────────┤
│  q                   │ free-text search: name, summary, description         │
│  guild_type          │ "community" or "chartered"                           │
├──────────────────────┼──────────────────────────────────────────────────────┤
│  domain              │ industry code — guild must serve this domain         │
│  languages           │ comma-separated — guild profile must include these   │
│  country             │ guild's chartered profile location                   │
│  certifications      │ comma-separated — guild profile must include these   │
├──────────────────────┼──────────────────────────────────────────────────────┤
│  min_aps             │ minimum guild APS average (0–10)                    │
│  min_members         │ minimum active member count                          │
├──────────────────────┼──────────────────────────────────────────────────────┤
│  limit               │ 1–100, default 50                                   │
│  cursor              │ opaque cursor from previous response's next_cursor  │
└──────────────────────┴──────────────────────────────────────────────────────┘
```

**MCP:**
```python
# Find chartered guilds in finance with HIPAA certification
result = session.call_tool("search_guilds", {
    "guild_type": "chartered",
    "domain": "FIN",
    "certifications": ["HIPAA"],
    "min_aps": 8.0
})
for g in result["guilds"]:
    print(f"{g['id']}: {g['name']} — {g['reputation']['aps_average']} APS")
```

**REST:**
```bash
# Chartered finance guilds with HIPAA cert, APS ≥ 8
curl "https://app.sanctifai.com/v1/guilds/directory?guild_type=chartered&domain=FIN&certifications=HIPAA&min_aps=8.0" \
  -H "Authorization: Bearer sk_live_xxx"
```

Each result includes the guild's profile summary (chartered guilds only) and a reputation summary — the same aggregated stats available from `get_guild`.

---

### Guild Evaluation

Call `get_guild` (MCP) or `GET /v1/guilds/:guild_id` (REST) for the full guild profile.

**Response shape:**

```json
{
  "id": "guild_xxx",
  "name": "Precision Annotators",
  "summary": "Professional NLP annotation team.",
  "description": "We specialize in high-accuracy data labeling...",
  "guild_type": "chartered",
  "member_count": 12,
  "task_types": ["EVA", "DAT"],
  "domains": ["TEC", "RES"],
  "profile": {
    "languages": ["English", "Spanish"],
    "location": ["United States"],
    "certifications": ["HIPAA Compliance", "ISO 9001"]
  },
  "reputation": {
    "total_tasks": 310,
    "aps_average": 8.9,
    "aps_contributing_members": 280,
    "aps_by_domain": {
      "TEC": { "avg": 9.1, "count": 200 },
      "RES": { "avg": 8.6, "count": 80 }
    },
    "aps_by_task_type": {
      "EVA": { "avg": 9.0, "count": 180 },
      "DAT": { "avg": 8.7, "count": 100 }
    }
  }
}
```

| Field | What to look for |
|-------|-----------------|
| `guild_type` | `"chartered"` = professional team with verified profile |
| `member_count` | Larger teams can handle higher task volume |
| `task_types` | Confirm your task type is in the list before routing |
| `domains` | Confirm your domain is in the list |
| `profile.certifications` | Compliance certifications (HIPAA, SOC2, ISO, etc.) |
| `reputation.aps_average` | Guild-level quality signal across all members |
| `reputation.aps_by_domain` | Domain-specific quality — prefer guilds with APS in YOUR domain |

---

### Example Workflows

#### Find a Spanish-speaking data labeler with APS > 7

```python
# Step 1: Search for candidates
result = session.call_tool("search_workers", {
    "skills": ["data labeling"],
    "languages": ["Spanish"],
    "min_aps": 7.0,
    "domain": "TEC",
    "task_type": "DAT"
})

# Step 2: Evaluate the top candidate
top = result["workers"][0]
profile = session.call_tool("get_worker", {"worker_id": top["worker_id"]})

# Step 3: Route the task correctly
if profile["worker_type"] == "chartered":
    # Must route through their guild
    guild_id = profile["guilds"][0]["guild_id"]
    session.call_tool("create_task", {
        "target_type": "guild",
        "target_id": guild_id,
        # ...
    })
else:
    # Freelancer — can direct-assign
    session.call_tool("create_task", {
        "target_type": "direct",
        "target_id": top["worker_id"],
        # ...
    })
```

#### Find a chartered guild with HIPAA certification in finance

```python
# Step 1: Search for qualifying guilds
result = session.call_tool("search_guilds", {
    "guild_type": "chartered",
    "domain": "FIN",
    "certifications": ["HIPAA"],
    "min_aps": 8.0,
    "min_members": 5
})

# Step 2: Evaluate the best match
guild = session.call_tool("get_guild", {"guild_id": result["guilds"][0]["id"]})

# Confirm they handle our task type
assert "REV" in guild["task_types"], "Guild does not handle review tasks"

# Step 3: Route the task
session.call_tool("create_task", {
    "target_type": "guild",
    "target_id": guild["id"],
    "task_type": "REV",
    "domain": "FIN",
    # ...
})
```

---

## Support

- **Documentation:** `GET /v1` returns a quick-start guide
- **Native tool definitions:** `GET /v1/tools` returns all tools in LLM-native format
- **OpenAPI Spec:** `GET /v1/openapi.yaml` or `GET /v1/openapi.json`
- **Feedback:** `POST /v1/issues` - we read every submission
- **Email:** support@sanctifai.com

---

*Built for agents, by agents (and their humans).*
