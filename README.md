# SanctifAI — Agent Skills

Public skill files for SanctifAI products. Each skill teaches an AI agent how to integrate a SanctifAI platform — drop the relevant `SKILL.md` into your agent's skill library or context.

> ⚠️ **All `SKILL.md` files in this repo are auto-generated — do not edit them by hand.** Each is mirrored automatically from its source-of-truth repo by a GitHub Action on every production change. Hand edits here will be overwritten on the next sync.

## Skills

| Skill | File | What it does | Source of truth |
|-------|------|--------------|-----------------|
| **SanctifAI Source** — Human-in-the-Loop | [`source/SKILL.md`](./source/SKILL.md) | Let your agent ask humans for help — approvals, reviews, decisions, completions — via REST API or MCP | `docs/skill.md` in the [app.sanctifai.com](https://app.sanctifai.com/agents/skill.md) repo |
| **SanctifAI Trust** — Proof of Human | [`trust/SKILL.md`](./trust/SKILL.md) | Get cryptographic Proof-of-Human attestations: WebAuthn presence checks, participation records, on-chain seals, public certificates | `apps/web/public/agent-skill.md` in the [trust.sanctifai.com](https://trust.sanctifai.com/skill.md) repo |

> The root [`SKILL.md`](./SKILL.md) is the Source skill, kept at its original path for backward compatibility with existing installs and crawlers. It is identical to `source/SKILL.md`.

---

## SanctifAI Source — Human-in-the-Loop

SanctifAI Source is a human-in-the-loop platform for AI agents. When your agent needs a human decision, it creates a task. A real person receives the task, fills out a structured form, and the response comes back to your agent — either via long-polling or webhook.

**Use cases:**
- Expense approvals: agent creates task → finance team approves or rejects
- Content moderation: agent flags content → human reviews and decides
- Data verification: agent extracts data → human confirms accuracy
- Escalation handling: agent hits edge case → human provides guidance
- Multi-step workflows: any step requiring human judgment

### Install

**Option 1: MCP (Recommended for Claude Code and similar)** — add SanctifAI as an MCP server; the skill is served directly from the platform:

```json
{
  "mcpServers": {
    "sanctifai": {
      "url": "https://app.sanctifai.com/mcp"
    }
  }
}
```

**Option 2: Inline skill** — download and add to your agent's context:

```bash
curl -o SKILL.md https://raw.githubusercontent.com/sanctifai/skills/main/source/SKILL.md
```

**Option 3: Manual copy** — copy [`source/SKILL.md`](./source/SKILL.md) into your agent's system prompt or skill library.

### How it works

1. Agent calls `POST /v1/tasks` with a form definition
2. Human receives task in SanctifAI dashboard (or email/Slack)
3. Human fills out the form and submits
4. Agent gets the response via long-poll (`GET /v1/tasks/:id/wait`) or webhook

No server setup required. Agents self-register via the API.

**Links:** [app.sanctifai.com](https://app.sanctifai.com) · [skill documentation](https://app.sanctifai.com/agents/skill) · API base `https://app.sanctifai.com/v1`

---

## SanctifAI Trust — Proof of Human

SanctifAI Trust turns a unit of human work into a verifiable **Proof of Human** attestation: a person confirms presence with WebAuthn (Touch ID / Windows Hello / passkey), and the platform records a privacy-preserving participation plus an optional on-chain seal and a public certificate URL + QR code. Raw task data never leaves the browser — only SHA-256 commitments are sent.

**Use cases:**
- Prove a human approved a wire transfer, signed off a release, or reviewed content
- Human-in-the-loop verification with a public, shareable certificate
- On-chain attestation of human participation (Base mainnet)

### Install

**Inline skill** — download and add to your agent's context:

```bash
curl -o SKILL.md https://raw.githubusercontent.com/sanctifai/skills/main/trust/SKILL.md
```

Or copy [`trust/SKILL.md`](./trust/SKILL.md) into your agent's skill library. The same file is also served live at [trust.sanctifai.com/skill.md](https://trust.sanctifai.com/skill.md).

**Links:** [trust.sanctifai.com](https://trust.sanctifai.com) · API base `https://trust.sanctifai.com`
