# SanctifAI — Agent Skills

Public skill files **and** the installable SanctifAI Trust plugin package. Product `SKILL.md` mirrors teach an AI agent how to integrate a SanctifAI platform. The Cursor/Claude/Codex plugin under `plugins/` is what you install in a harness.

```
sanctifai/skills
├── source/SKILL.md              auto-mirrored product skill (Source HITL)
├── trust/SKILL.md               auto-mirrored product skill (Trust: all 3 surfaces)
└── plugins/sanctifai-trust/     installable Chat-bridge plugin (this repo)
```

> ⚠️ **`source/SKILL.md` and `trust/SKILL.md` are auto-generated — do not edit them by hand.** Each is mirrored from its source-of-truth repo by a GitHub Action on every production change. Hand edits there will be overwritten on the next sync.
>
> `plugins/sanctifai-trust/` is the public plugin package (not the auto-sync product skills). The Chat bridge **runtime** stays in private `sanctifai/sanctifai-trust` (`apps/chat-bridge`) at `https://bridge.trust.sanctifai.com`.

## Skills (auto-mirrored)

| Skill | File | What it does | Source of truth |
|-------|------|--------------|-----------------|
| **SanctifAI Source** — Human-in-the-Loop | [`source/SKILL.md`](./source/SKILL.md) | Let your agent ask humans for help — approvals, reviews, decisions, completions — via REST API or MCP | `docs/skill.md` in the [app.sanctifai.com](https://app.sanctifai.com/agents/skill.md) repo |
| **SanctifAI Trust** — Proof of Human | [`trust/SKILL.md`](./trust/SKILL.md) | Get cryptographic Proof-of-Human attestations: WebAuthn presence checks, participation records, on-chain seals, public certificates | `apps/web/public/agent-skill.md` in the [trust.sanctifai.com](https://trust.sanctifai.com/skill.md) repo |

> Each product skill lives only in its subfolder. The former root `SKILL.md` (the Source skill's original path) was retired on 2026-07-08 — if you fetched that URL, switch to [`source/SKILL.md`](./source/SKILL.md).

## Cursor plugin package

| Plugin | Path | What it does |
|--------|------|--------------|
| **sanctifai-trust** | [`plugins/sanctifai-trust`](./plugins/sanctifai-trust) | Installable Cursor / Claude Code / Codex / Agent plugin for **Chat bridge** Proof of Human. Bundled skill: [`skills/proof-of-human/SKILL.md`](./plugins/sanctifai-trust/skills/proof-of-human/SKILL.md). Default bridge: `https://bridge.trust.sanctifai.com`. |

This repo is a **multi-plugin marketplace** layout (`.cursor-plugin/marketplace.json`). Only Trust is listed today. A **Source** sibling plugin may be added later in `plugins/` and registered in the same marketplace file — do not add a placeholder entry until that package exists.

Cursor reads `.cursor-plugin/marketplace.json` (not a root `marketplace.json`). Plugin identity lives in `plugins/sanctifai-trust/.cursor-plugin/plugin.json` plus the portable `plugin.json` / `.claude-plugin/` / `.codex-plugin/` manifests.

This repository is **not** submitted to the public [Cursor Marketplace](https://cursor.com/marketplace). Do not publish it there.

## License

This repository is licensed under the [Apache License 2.0](./LICENSE) (copyright 2026 SanctifAI). See [NOTICE](./NOTICE) for trademarks, patent reservation (including App. 63/926,453), and license **scope**: Apache-2.0 covers this skills/plugin packaging repo only. It does **not** grant trademark rights, it does **not** grant patents beyond Apache-2.0 §3, and it does **not** license the Trust SaaS/service or proprietary product/runtime code (the Chat bridge runtime stays private).

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

### Product skill (inline)

Download and add to your agent's context:

```bash
curl -o SKILL.md https://raw.githubusercontent.com/sanctifai/skills/main/trust/SKILL.md
```

Or copy [`trust/SKILL.md`](./trust/SKILL.md) into your agent's skill library. The same file is also served live at [trust.sanctifai.com/skill.md](https://trust.sanctifai.com/skill.md). Covers all three surfaces: Embedded, Chrome extension, and Chat bridge.

**Links:** [trust.sanctifai.com](https://trust.sanctifai.com) · API base `https://trust.sanctifai.com`

### Chat bridge plugin

When the human is in an AI chat with no `navigator.credentials`, install [`plugins/sanctifai-trust`](./plugins/sanctifai-trust) instead of (or in addition to) the product skill. Default `APP_BASE_URL`: `https://bridge.trust.sanctifai.com`. See that plugin's [README](./plugins/sanctifai-trust/README.md).
