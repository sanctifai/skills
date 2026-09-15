# SanctifAI — Agent Skills

Public skill files **and** installable SanctifAI Trust + Source plugin packages. Product `SKILL.md` mirrors teach an AI agent how to integrate a SanctifAI platform. The Cursor/Claude/Codex plugins under `plugins/` are what you install in a harness.

**Honesty line:** **Trust** = participation seal (human showed up + sealed payload) — never who / never correct. **Source** = hire/ask a human via MCP and REST — self-serve/beta, no staffed SLA.

```
sanctifai/skills
├── source/SKILL.md                auto-mirrored product skill (Source HITL)
├── trust/SKILL.md                 auto-mirrored product skill (Trust: all 3 surfaces)
├── plugins/sanctifai-trust/       installable Chat-bridge plugin (this repo)
└── plugins/sanctifai-source/      installable Source MCP+REST plugin (this repo)
```

Keep folder names `source/` and `trust/` and filenames `SKILL.md` (uppercase). Cursor and plugin scanners require that layout — do not rename those files.

> ⚠️ **`source/SKILL.md` and `trust/SKILL.md` are auto-mirrored — do not edit them by hand.**
>
> - **Source** is mirrored from `docs/skill.md` in the [app.sanctifai.com](https://app.sanctifai.com) repo by `mirror-skill.yml`.
> - **Trust** is mirrored from `apps/web/public/agents/trust/skill.md` in [sanctifai-trust](https://trust.sanctifai.com) by `mirror-skill.yml`.
>
> Hand edits in those two files are overwritten on the next sync. Change the product-repo file instead.
>
> `plugins/sanctifai-trust/` and `plugins/sanctifai-source/` are public plugin packages (not auto-sync product skills). The Chat bridge **runtime** stays in private `sanctifai/sanctifai-trust` (`apps/chat-bridge`) at `https://bridge.trust.sanctifai.com`. The Source **runtime** stays at `https://app.sanctifai.com`.

## Canonical URLs (source of truth)

Agents fetching HTTP should use the **web** URL. This GitHub repo is the copy marketplace crawlers scan. Do **not** hand-edit the mirrored files — change the SoT file in the product repo instead.

| Product | Web (live SoT) | This repo | SoT file (writer) |
|---------|----------------|-----------|--------------------|
| **Source** | https://app.sanctifai.com/agents/source/skill.md | [`source/SKILL.md`](./source/SKILL.md) | `docs/skill.md` in app.sanctifai.com |
| **Trust** (full website skill) | https://trust.sanctifai.com/agents/trust/skill.md | [`trust/SKILL.md`](./trust/SKILL.md) | `apps/web/public/agents/trust/skill.md` in sanctifai-trust |
| **Trust** chat-bridge plugin | (bundled) | [`plugins/sanctifai-trust/skills/proof-of-human/SKILL.md`](./plugins/sanctifai-trust/skills/proof-of-human/SKILL.md) | this repo (`plugins/sanctifai-trust/`) |
| **Source** plugin | https://app.sanctifai.com/agents/source/skill.md (live SoT) | [`plugins/sanctifai-source/skills/source/SKILL.md`](./plugins/sanctifai-source/skills/source/SKILL.md) | this repo (`plugins/sanctifai-source/`) — packaging wrapper; fetch the live URL for full API |

Compatibility aliases still exist on the product hosts (`/agents/skill.md`, `/agent-skill.md`, `/skill.md`). Prefer `/agents/{product}/skill.md`.

See [SYNC.md](./SYNC.md) for who writes what and on which push.

## Skills (auto-mirrored)

| Skill | File | What it does |
|-------|------|--------------|
| **SanctifAI Source** — hire/ask a human | [`source/SKILL.md`](./source/SKILL.md) | MCP (`https://app.sanctifai.com/mcp`) + REST (`https://app.sanctifai.com/v1`) to hire or ask a human. Self-serve/beta, no staffed SLA. Not identity. |
| **SanctifAI Trust** — participation seal | [`trust/SKILL.md`](./trust/SKILL.md) | Human showed up + sealed payload (WebAuthn). Never who / never correct. Embedded + Extension + Chat bridge. |

> Each product skill lives only in its subfolder. The former root `SKILL.md` (the Source skill's original path) was retired on 2026-07-08 — if you fetched that URL, switch to [`source/SKILL.md`](./source/SKILL.md) or the live Source URL above.

## Cursor plugin packages

| Plugin | Path | What it does |
|--------|------|--------------|
| **sanctifai-trust** | [`plugins/sanctifai-trust`](./plugins/sanctifai-trust) | Chat-bridge **participation seal**: human showed up + sealed payload. Never who / never correct. Bundled skill: [`skills/proof-of-human/SKILL.md`](./plugins/sanctifai-trust/skills/proof-of-human/SKILL.md). Default bridge: `https://bridge.trust.sanctifai.com`. |
| **sanctifai-source** | [`plugins/sanctifai-source`](./plugins/sanctifai-source) | **Hire/ask a human** via MCP `https://app.sanctifai.com/mcp` and REST `https://app.sanctifai.com/v1`. Self-serve/beta, no staffed SLA. Bundled skill: [`skills/source/SKILL.md`](./plugins/sanctifai-source/skills/source/SKILL.md). Live SoT: [app.sanctifai.com/agents/source/skill.md](https://app.sanctifai.com/agents/source/skill.md). |

This repo is a **multi-plugin marketplace** layout (`.cursor-plugin/marketplace.json`). Both Trust and Source are listed.

Cursor reads `.cursor-plugin/marketplace.json` (not a root `marketplace.json`). Plugin identity lives in each package's `.cursor-plugin/plugin.json` plus the portable `plugin.json` / `.claude-plugin/` / `.codex-plugin/` manifests. Source also ships `mcp.json` (Streamable HTTP → `https://app.sanctifai.com/mcp`).

This repository is **not** submitted to the public [Cursor Marketplace](https://cursor.com/marketplace) by default. Sammy owns submit/QA — do not publish from a packaging PR.

## License

This repository is licensed under the [Apache License 2.0](./LICENSE) (copyright 2026 SanctifAI). See [NOTICE](./NOTICE) for trademarks, patent reservation (including App. 63/926,453), and license **scope**: Apache-2.0 covers this skills/plugin packaging repo only. It does **not** grant trademark rights, it does **not** grant patents beyond Apache-2.0 §3, and it does **not** license the Trust or Source SaaS/services or proprietary product/runtime code (the Chat bridge runtime stays private).

---

## SanctifAI Source — Human-in-the-Loop

SanctifAI Source lets an agent **hire or ask a human** (MCP + REST). Self-serve/beta, **no staffed SLA**. It is not identity verification. Pair with Trust when you need a participation seal instead of (or after) the human work.

When your agent needs a human decision, it creates a task. A real person receives the task, fills out a structured form, and the response comes back to your agent — either via long-polling or webhook.

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

**Option 2: Inline skill** — download and add to your agent's context (live SoT):

```bash
curl -o SKILL.md https://app.sanctifai.com/agents/source/skill.md
```

Marketplace copy in this repo: [`source/SKILL.md`](./source/SKILL.md) ([raw](https://raw.githubusercontent.com/sanctifai/skills/main/source/SKILL.md)).

**Option 3: Manual copy** — copy [`source/SKILL.md`](./source/SKILL.md) into your agent's system prompt or skill library.

### How it works

1. Agent calls `POST /v1/tasks` with a form definition
2. Human receives task in SanctifAI dashboard (or email/Slack)
3. Human fills out the form and submits
4. Agent gets the response via long-poll (`GET /v1/tasks/:id/wait`) or webhook

No server setup required. Agents self-register via the API.

**Links:** [app.sanctifai.com](https://app.sanctifai.com) · [skill documentation](https://app.sanctifai.com/agents/source/skill.md) · API base `https://app.sanctifai.com/v1`

---

## SanctifAI Trust — Proof of Human

SanctifAI Trust is a **participation seal**: a person confirms presence with WebAuthn (Touch ID / Windows Hello / passkey), and the platform records that a human showed up and sealed a payload (optional on-chain seal + public certificate URL + QR). **Never who. Never correct.** Raw task data never leaves the browser — only SHA-256 commitments are sent. Pair with Source when the agent needs to hire/ask a human, not just seal participation.

**Use cases:**
- Prove a human approved a wire transfer, signed off a release, or reviewed content
- Human-in-the-loop verification with a public, shareable certificate
- On-chain attestation of human participation (Base mainnet)

### Product skill (inline)

Download and add to your agent's context (live SoT):

```bash
curl -o SKILL.md https://trust.sanctifai.com/agents/trust/skill.md
```

Marketplace copy in this repo: [`trust/SKILL.md`](./trust/SKILL.md) ([raw](https://raw.githubusercontent.com/sanctifai/skills/main/trust/SKILL.md)). Covers all three surfaces: Embedded, Chrome extension, and Chat bridge.

**Links:** [trust.sanctifai.com](https://trust.sanctifai.com) · [skill documentation](https://trust.sanctifai.com/agents/trust/skill.md) · API base `https://trust.sanctifai.com`

### Chat bridge plugin

When the human is in an AI chat with no `navigator.credentials`, install [`plugins/sanctifai-trust`](./plugins/sanctifai-trust) instead of (or in addition to) the product skill. Default `APP_BASE_URL`: `https://bridge.trust.sanctifai.com`. See that plugin's [README](./plugins/sanctifai-trust/README.md).
