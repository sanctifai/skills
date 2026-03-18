# SanctifAI — Human-in-the-Loop Skill for AI Agents

Give your AI agent the ability to ask humans for help — approvals, reviews, decisions, and completions — with a simple API or MCP connection.

## What It Does

SanctifAI is a human-in-the-loop platform for AI agents. When your agent needs a human decision, it creates a task. A real person receives the task, fills out a structured form, and the response comes back to your agent — either via long-polling or webhook.

**Use cases:**
- Expense approvals: agent creates task → finance team approves or rejects
- Content moderation: agent flags content → human reviews and decides
- Data verification: agent extracts data → human confirms accuracy
- Escalation handling: agent hits edge case → human provides guidance
- Multi-step workflows: any step requiring human judgment

## Install

### Option 1: MCP (Recommended for Claude Code and similar)

Add SanctifAI as an MCP server. The skill file is served directly from the platform:

```json
{
  "mcpServers": {
    "sanctifai": {
      "url": "https://app.sanctifai.com/mcp"
    }
  }
}
```

### Option 2: Inline Skill (Claude Code)

Download `SKILL.md` and add it to your agent's context:

```bash
curl -o SKILL.md https://raw.githubusercontent.com/sanctifai/skills/main/SKILL.md
```

Then reference it in your CLAUDE.md or system prompt.

### Option 3: Manual Copy

Copy the contents of [`SKILL.md`](./SKILL.md) directly into your agent's system prompt or skill library.

## Links

- **Platform:** [app.sanctifai.com](https://app.sanctifai.com)
- **Skill documentation:** [app.sanctifai.com/agents/skill](https://app.sanctifai.com/agents/skill)
- **API base URL:** `https://app.sanctifai.com/v1`

## How It Works

1. Agent calls `POST /v1/tasks` with a form definition
2. Human receives task in SanctifAI dashboard (or email/Slack)
3. Human fills out the form and submits
4. Agent gets the response via long-poll (`GET /v1/tasks/:id/wait`) or webhook

No server setup required. Agents self-register via the API.
