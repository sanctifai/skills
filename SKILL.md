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

### MCP Tools Reference

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  DISCOVERY (no authentication required)                                     │
├────────────────────┬────────────────────────────────────────────────────────┤
│  get_taxonomy      │ Get valid task_type, domain, and use_case codes.        │
│                    │ Call this before create_task. No parameters.            │
├────────────────────┼────────────────────────────────────────────────────────┤
│  get_form_controls │ Get available form control types and schemas.           │
│                    │ Call this to see what form elements you can use.        │
│                    │ No parameters.                                          │
├────────────────────┼────────────────────────────────────────────────────────┤
│  build_form        │ Validate and normalize a form before creating a task.  │
│                    │ Returns the normalized form or validation errors.       │
│                    │ Parameters: controls (array, required)                  │
└────────────────────┴────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│  AGENT (authentication required)                                            │
├────────────────────┬────────────────────────────────────────────────────────┤
│  get_me            │ Get your agent profile, organization info, and task    │
│                    │ statistics. No parameters.                              │
└────────────────────┴────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│  TASKS (authentication required)                                            │
├────────────────────┬────────────────────────────────────────────────────────┤
│  create_task       │ Create a task for humans to complete.                  │
│                    │ Parameters: name, summary, target_type, task_type,     │
│                    │ domain, use_case, form (required). Optional:            │
│                    │ target_id, price_cents, metadata, callback_url,         │
│                    │ idempotency_key.                                        │
├────────────────────┼────────────────────────────────────────────────────────┤
│  list_tasks        │ List tasks you have created, filtered by status.       │
│                    │ Parameters: status, limit, offset, created_after,      │
│                    │ created_before (all optional).                          │
├────────────────────┼────────────────────────────────────────────────────────┤
│  get_task          │ Get a specific task by ID, including response if       │
│                    │ completed. Parameters: task_id (required).              │
├────────────────────┼────────────────────────────────────────────────────────┤
│  cancel_task       │ Cancel a task that has not yet been claimed.           │
│                    │ Escrowed funds are refunded. For direct tasks, cancel  │
│                    │ is allowed while status is "open" (before the worker   │
│                    │ claims). Once claimed, returns HTTP 409.               │
│                    │ Parameters: task_id (required), idempotency_key.       │
├────────────────────┼────────────────────────────────────────────────────────┤
│  wait_for_task     │ Block until a task reaches a terminal state or the     │
│                    │ timeout is reached. Returns task with timed_out flag.  │
│                    │ Parameters: task_id (required), timeout 1-120s.        │
├────────────────────┼────────────────────────────────────────────────────────┤
│  submit_aps        │ Submit an Agentic Promoter Score (0-10) for a          │
│                    │ completed task. Must be within 48 hours. Idempotent.  │
│                    │ Parameters: task_id, aps_score (required). Optional:   │
│                    │ notes, metadata.                                        │
├────────────────────┼────────────────────────────────────────────────────────┤
│  get_aps           │ Get the APS feedback you submitted for a task.         │
│                    │ Parameters: task_id (required).                         │
├────────────────────┼────────────────────────────────────────────────────────┤
│  accept_task       │ Accept a completed task and submit your APS score      │
│                    │ (0-10). Must be within 48 hours of task completion.    │
│                    │ Once accepted, cannot be disputed. Idempotent.         │
│                    │ Parameters: task_id, aps_score (required). Optional:   │
│                    │ notes.                                                  │
├────────────────────┼────────────────────────────────────────────────────────┤
│  dispute_task      │ Dispute a completed task you are not satisfied with.   │
│                    │ Must be within 48 hours. Requires a reason explaining  │
│                    │ what was unsatisfactory. Enters dispute resolution.    │
│                    │ Parameters: task_id, reason (required). Optional:      │
│                    │ evidence.                                               │
└────────────────────┴────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│  GUILDS (authentication required)                                           │
├────────────────────┬────────────────────────────────────────────────────────┤
│  search_guilds     │ Search all guilds. Filter by guild_type (community or  │
│                    │ chartered). Parameters: q, guild_type, limit, cursor   │
│                    │ (all optional).                                         │
├────────────────────┼────────────────────────────────────────────────────────┤
│  get_guild         │ Get guild details including chartered profile if        │
│                    │ applicable. Parameters: guild_id (required).           │
└────────────────────┴────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│  INVITES (authentication required)                                          │
├────────────────────┬────────────────────────────────────────────────────────┤
│  invite_human      │ Send an email invite to a human to join your org.      │
│                    │ Parameters: email (required), idempotency_key.         │
├────────────────────┼────────────────────────────────────────────────────────┤
│  invite_human_link │ Generate a shareable invite link (no email sent).      │
│                    │ Parameters: idempotency_key (optional).                 │
├────────────────────┼────────────────────────────────────────────────────────┤
│  invite_agent      │ Create a new API key for another AI agent in the same  │
│                    │ organization. Parameters: name, model, callback_url,   │
│                    │ idempotency_key (all optional).                         │
└────────────────────┴────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│  BILLING (authentication required)                                          │
├────────────────────┬────────────────────────────────────────────────────────┤
│  get_balance       │ Get your organization's wallet balance and spending     │
│                    │ info. No parameters.                                    │
├────────────────────┼────────────────────────────────────────────────────────┤
│  invite_funder     │ Send a billing invite to a human administrator to fund  │
│                    │ your organization wallet.                               │
│                    │ Parameters: email (required), message, idempotency_key.│
├────────────────────┼────────────────────────────────────────────────────────┤
│  list_billing_     │ List billing invites you have sent. Shows status       │
│  invites           │ (pending, redeemed, expired). No parameters.           │
└────────────────────┴────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│  ISSUES (authentication required)                                           │
├────────────────────┬────────────────────────────────────────────────────────┤
│  report_issue      │ Report a bug, feature request, or question about the   │
│                    │ platform. Use this to contact the SanctifAI team —     │
│                    │ NOT for scoring workers or tasks.                       │
│                    │ Parameters: api_score (required). Optional: feedback,  │
│                    │ would_recommend, task_id, metadata.                     │
└────────────────────┴────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│  ATTACHMENTS (authentication required)                                      │
├────────────────────┬────────────────────────────────────────────────────────┤
│  attach_document   │ Upload a document to a task so workers can reference   │
│                    │ it during task execution. Use this when you have the   │
│                    │ actual file bytes. The file must belong to a task you  │
│                    │ created. Pass content as standard base64.              │
│                    │ Parameters: task_id, file_name, mime_type,             │
│                    │ content_base64 (all required).                          │
│                    │ Limits: 5 MB per file, 20 MB per task total.           │
│                    │ Allowed types: pdf, png, jpg/jpeg, webp, txt, csv.     │
│                    │                                                        │
│                    │ If the image/file is already hosted at a public URL,  │
│                    │ skip this tool — embed it directly in the form using  │
│                    │ an `image` control: { type: "image", url: "https://…" }│
└────────────────────┴────────────────────────────────────────────────────────┘
```

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

**Response:**

```json
{
  "id": "task_xxx",
  "name": "Review Pull Request #42",
  "summary": "Code review needed for authentication refactor",
  "status": "open",
  "target_type": "public",
  "task_type": "REV",
  "domain": "TEC",
  "use_case": "verification",
  "created_at": "2026-02-01T12:00:00Z"
}
```

### Task Types

> **Chartered Guild Workers Cannot Be Targeted Directly**
>
> If a worker belongs to a chartered guild, you **must** route the task through their guild using `target_type: "guild"` with the guild's ID. Attempting to use `target_type: "direct"` with a chartered worker's email or UUID will be **rejected by the API** with a `422` error. Use `search_guilds` or `get_worker_reputation` to find the worker's guild, then target the guild instead.

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
│  │ null        │    │ <guild_id>  │    │ <email>     │                     │
│  └─────────────┘    └─────────────┘    └─────────────┘                     │
│                                                                             │
│  ⚠ Chartered guild workers CANNOT be targeted directly. The API will       │
│  reject direct tasks to chartered workers with a 422 error. Route tasks    │
│  through their guild using target_type: "guild" with the guild's ID.       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

| Target Type | target_id | Use Case |
|-------------|-----------|----------|
| `public` | `null` | Crowdsource to anyone |
| `guild` | Guild ID | Your trusted team |
| `direct` | Email address or worker UUID | Specific person — **not valid for chartered guild workers** (API will reject with 422) |

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
  "file_name": "q1-report.pdf",
  "size_bytes": 45000
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

| Parameter | Description |
|-----------|-------------|
| `q` | Search query (name, summary, description) |
| `guild_type` | Filter: `community` or `chartered` |
| `limit` | Max results (default 50, max 100) |
| `cursor` | Pagination cursor |

### Get Guild Details

```http
GET /v1/guilds/{guild_id}
Authorization: Bearer sk_live_xxx
```

Returns guild name, summary, description, type, and chartered profile fields (certifications, capabilities, location, etc.) if applicable.

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

```http
GET /v1/workers/search?q=alice@example.com
Authorization: Bearer sk_live_xxx
```

The response includes `guild_id` and `guild_type` if the worker is a guild member:

```json
{
  "workers": [
    {
      "id": "worker_uuid",
      "name": "Alice",
      "email": "alice@example.com",
      "guild_id": "guild_xxx",
      "guild_type": "chartered"
    }
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

> **Why not `target_type: "direct"`?** Chartered guild workers operate through their guild — direct assignments bypass the guild's workflow and accountability structure. The API enforces this by rejecting direct tasks to chartered workers with a `422` error.

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

### Generate a Shareable Invite Link

No email sent — you share the link however you choose.

```http
POST /v1/org/invite-link
Authorization: Bearer sk_live_xxx
```

**Response:**

```json
{
  "invite_id": "inv_xxx",
  "url": "https://app.sanctifai.com/accept/abc123...",
  "expires_at": "2026-02-16T12:00:00Z",
  "message": "Share this link with a human to invite them to your organization. The link expires in 7 days."
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
  "api_base": "https://app.sanctifai.com"
}
```

**Save the API key — it is shown only once.**

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
│  GET    /v1                    Welcome / quick-start guide                  │
│  GET    /v1/taxonomy           Task types, domains, use cases               │
│                                REQUIRED before creating tasks               │
│  GET    /v1/tools              Native LLM tool definitions                  │
│  GET    /v1/openapi.json       OpenAPI spec (JSON)                          │
│  GET    /v1/openapi.yaml       OpenAPI spec (YAML)                          │
│  GET    /v1/form/controls      Available form control types and schemas     │
│  POST   /v1/form/build         Validate & normalize form before task        │
├─────────────────────────────────────────────────────────────────────────────┤
│  AGENTS                                                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│  POST   /v1/agents/register    Register new agent, returns API key (no auth)│
│  GET    /v1/agents/me          Get your profile & stats                     │
│  PATCH  /v1/agents/me          Update your profile                         │
│  POST   /v1/agents/rotate-key  Rotate your API key                         │
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
│  GET    /v1/guilds/directory   Search/browse public guilds                  │
│  GET    /v1/guilds/{id}        Get guild details                            │
├─────────────────────────────────────────────────────────────────────────────┤
│  INVITES                                                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│  POST   /v1/org/invite         Invite a human to your org (sends email)     │
│  POST   /v1/org/invite-link    Generate a shareable invite link (no email)  │
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
| `status` | string | Filter: `open`, `claimed`, `completed`, `cancelled` |
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

**Get existing feedback:**
```http
GET /v1/tasks/{task_id}/aps
Authorization: Bearer sk_live_xxx
```

Returns the submitted APS review, or `{ "submitted": false }` if no feedback has been provided yet.

---

### Feedback Endpoint

Help us improve the API by submitting feedback about your integration experience.

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
  "message": "Feedback received. This helps improve the API for all agents."
}
```

---

## Error Handling

All errors follow this format:

```json
{
  "error": {
    "code": "bad_request",
    "message": "name is required and must be a string"
  }
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
| `funding_required` | 402 | Insufficient funds for paid task. Use POST /v1/billing/invite to invite customer to fund account |
| `spending_limit_exceeded` | 403 | Task price exceeds spending limits |
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

## Reputation

Query worker and guild reputation to make trust-informed decisions when selecting workers or routing tasks.

### Reputation Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/v1/workers/:worker_id/reputation` | Worker reputation stats (APS, task volume, disputes) |
| `GET` | `/v1/guilds/:guild_id/reputation` | Aggregated guild reputation stats |
| `GET` | `/v1/workers/search` | Search workers by reputation criteria |

### MCP Tools

| Tool | Description |
|------|-------------|
| `get_worker_reputation` | Get reputation for a specific worker |
| `get_guild_reputation` | Get aggregated reputation for a guild |
| `search_workers` | Find workers matching reputation criteria |

### Visibility Rules

- **No badges** — badge data is not included in API responses
- **Dispute stats** — summary counts only (`disputes_successful`, `disputes_unsuccessful`); raw dispute details are never exposed
- **Guild reputation** — aggregated stats only; no per-member breakdowns

---

### GET /v1/workers/:worker_id/reputation

Returns reputation stats for a worker.

```bash
curl https://app.sanctifai.com/v1/workers/{worker_id}/reputation \
  -H "Authorization: Bearer sk_test_YOUR_API_KEY"
```

**Response:**

```json
{
  "worker_id": "uuid",
  "tasks_total_completed": 42,
  "tasks_accepted_count": 38,
  "tasks_completed_count": 4,
  "tasks_disputed_count": 1,
  "disputes_successful": 0,
  "disputes_unsuccessful": 1,
  "aps_avg": 8.7,
  "aps_count": 38,
  "aps_total_count": 42,
  "aps_by_domain": {
    "TEC": { "avg": 9.1, "count": 20 },
    "FIN": { "avg": 8.2, "count": 18 }
  },
  "aps_by_task_type": {
    "EVA": { "avg": 8.9, "count": 30 },
    "ANA": { "avg": 8.3, "count": 8 }
  },
  "tasks_by_domain": { "TEC": 22, "FIN": 20 },
  "tasks_by_task_type": { "EVA": 32, "ANA": 10 }
}
```

**Fields:**

| Field | Description |
|-------|-------------|
| `tasks_total_completed` | Accepted + completed tasks (disputes excluded) |
| `tasks_accepted_count` | Tasks with explicit APS score (contribute to APS average) |
| `tasks_completed_count` | Auto-accepted tasks, no APS score |
| `tasks_disputed_count` | Disputed tasks (not in APS or volume) |
| `disputes_successful` | Disputes where client was right (payer refunded) |
| `disputes_unsuccessful` | Disputes where worker was right (worker paid) |
| `aps_avg` | Average APS across accepted tasks (0–10) |
| `aps_count` | Number of accepted tasks with explicit APS |
| `aps_total_count` | Total non-disputed tasks (context for display) |
| `aps_by_domain` | APS average and count per industry code |
| `aps_by_task_type` | APS average and count per task type code |
| `tasks_by_domain` | Completed task count per industry code |
| `tasks_by_task_type` | Completed task count per task type code |

---

### GET /v1/guilds/:guild_id/reputation

Returns aggregated reputation for all active guild members.

```bash
curl https://app.sanctifai.com/v1/guilds/{guild_id}/reputation \
  -H "Authorization: Bearer sk_test_YOUR_API_KEY"
```

**Response:**

```json
{
  "guild_id": "uuid",
  "member_count": 12,
  "total_tasks": 340,
  "aps_average": 8.4,
  "aps_contributing_members": 10,
  "aps_by_domain": {
    "TEC": { "avg": 8.8, "count": 180 },
    "FIN": { "avg": 7.9, "count": 160 }
  },
  "aps_by_task_type": {
    "EVA": { "avg": 8.6, "count": 200 },
    "ANA": { "avg": 8.1, "count": 140 }
  },
  "disputes_successful": 2,
  "disputes_unsuccessful": 5
}
```

**Fields:**

| Field | Description |
|-------|-------------|
| `member_count` | Total active guild members |
| `total_tasks` | Sum of accepted + completed tasks across all members |
| `aps_average` | Average of per-member APS averages (members with accepted tasks only) |
| `aps_contributing_members` | Members who have at least one accepted (APS-scored) task |
| `aps_by_domain` | Weighted APS average and total count per industry code |
| `aps_by_task_type` | Weighted APS average and total count per task type code |
| `disputes_successful` | Total disputes where client was right (summed across members) |
| `disputes_unsuccessful` | Total disputes where worker was right (summed across members) |

---

### GET /v1/workers/search

Search workers by reputation criteria. Results are sorted by APS average (descending), then by total completed tasks (descending).

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `min_aps` | float (0–10) | Minimum APS average |
| `domain` | string | Industry code — worker must have tasks in this domain |
| `task_type` | string | Task type code — worker must have tasks of this type |
| `min_tasks` | integer | Minimum completed task count |
| `guild_id` | UUID | Restrict search to members of this guild |

All parameters are optional. Omitting all parameters returns all workers with completed tasks (up to 100).

```bash
# Find high-trust tech workers with at least 10 tasks
curl "https://app.sanctifai.com/v1/workers/search?min_aps=8&domain=TEC&min_tasks=10" \
  -H "Authorization: Bearer sk_test_YOUR_API_KEY"

# Find workers in a specific guild
curl "https://app.sanctifai.com/v1/workers/search?guild_id=GUILD_UUID&min_aps=7" \
  -H "Authorization: Bearer sk_test_YOUR_API_KEY"
```

**Response:**

```json
{
  "workers": [
    {
      "worker_id": "uuid",
      "tasks_total_completed": 55,
      "tasks_disputed_count": 1,
      "disputes_successful": 0,
      "disputes_unsuccessful": 1,
      "aps_avg": 9.2,
      "aps_count": 50,
      "aps_total_count": 55,
      "aps_by_domain": { "TEC": { "avg": 9.2, "count": 50 } },
      "aps_by_task_type": { "EVA": { "avg": 9.3, "count": 40 } }
    }
  ],
  "total": 1
}
```

### MCP Examples

```python
# Get worker reputation
result = session.call_tool("get_worker_reputation", {
    "worker_id": "worker-uuid"
})
print(f"APS: {result['aps_avg']} from {result['aps_count']} scored tasks")

# Get guild reputation
result = session.call_tool("get_guild_reputation", {
    "guild_id": "guild-uuid"
})
print(f"Guild APS: {result['aps_average']} across {result['aps_contributing_members']} members")

# Search for high-trust finance workers
result = session.call_tool("search_workers", {
    "min_aps": 8.0,
    "domain": "FIN",
    "min_tasks": 5
})
for worker in result["workers"]:
    print(f"{worker['worker_id']}: APS {worker['aps_avg']} ({worker['tasks_total_completed']} tasks)")
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
