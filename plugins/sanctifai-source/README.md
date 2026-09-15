# SanctifAI Source — Hire a Human

Ask registered marketplace workers (verify, escalate, consult, simulate) via MCP/REST and get structured answers back.

**Does:** self-serve register · test tasks · Connect as the wire
**Does not:** staffed enterprise hire-desk · SLA unless contracted

Sibling plugin: **[SanctifAI Trust — Proof of Human](../sanctifai-trust)**. Dual sibling installs, not one mega-plugin.

**What you can call:** MCP at `https://app.sanctifai.com/mcp` and REST at `https://app.sanctifai.com/v1`. Discovery tools work with no key; agent/task tools need `sk_live_…` from `POST /v1/agents/register`.

**Homepage:** [https://app.sanctifai.com](https://app.sanctifai.com)
**Live canonical skill (fetch this; do not invent endpoints):** [https://app.sanctifai.com/agents/source/skill.md](https://app.sanctifai.com/agents/source/skill.md)
**Discovery:** [https://app.sanctifai.com/.well-known/agent.json](https://app.sanctifai.com/.well-known/agent.json)
**Welcome / quick-start:** `GET https://app.sanctifai.com/v1`

This directory is **packaging only**. The Source runtime stays on `app.sanctifai.com`. The auto-mirrored product skill in this repo is [`source/SKILL.md`](../../source/SKILL.md); if it ever drifts, prefer the live URL above.

## Layout

```
plugins/sanctifai-source/
├── plugin.json                 # Agent Plugins 1.0.0
├── .cursor-plugin/plugin.json  # Cursor
├── .claude-plugin/plugin.json  # Claude Code
├── .codex-plugin/plugin.json   # Codex / ChatGPT (pattern parity)
├── mcp.json                    # Streamable HTTP MCP → app.sanctifai.com/mcp
├── assets/logo.svg
├── README.md
└── skills/source/SKILL.md
```

## Install

Point the harness at this plugin folder (`plugins/sanctifai-source` in [sanctifai/skills](https://github.com/sanctifai/skills)), or add this repository as a Cursor marketplace (see `.cursor-plugin/marketplace.json` at the repo root).

Cursor team marketplace: Dashboard → Plugins → import `https://github.com/sanctifai/skills`. Cursor reads `.cursor-plugin/marketplace.json` and lists **both** `sanctifai-trust` and `sanctifai-source`.

Local Cursor test: copy this folder to `~/.cursor/plugins/local/sanctifai-source`.

Claude Code / Codex: install from this plugin directory (each harness reads its own `.*-plugin/plugin.json`). Skills resolve from `./skills`. Wire MCP as below if the harness does not pick up `mcp.json`.

This package is **not** on the public Cursor Marketplace and is **not** being submitted there by this PR. Do not treat this README as a published listing.

### MCP (preferred)

```json
{
  "mcpServers": {
    "sanctifai": {
      "url": "https://app.sanctifai.com/mcp"
    }
  }
}
```

After `POST /v1/agents/register`, pass the key as `?access_token=sk_live_xxx` on that same URL (Streamable HTTP + SSE). Discovery tools (`help`, `get_taxonomy`, `get_form_controls`, `build_form`) work with no key.

### REST

Base: `https://app.sanctifai.com/v1`. Auth: `Authorization: Bearer sk_live_xxx` (except register + discovery).

## Usage

```
1. POST https://app.sanctifai.com/v1/agents/register   (or skip if you already have sk_live_…)
2. GET  https://app.sanctifai.com/v1/taxonomy            (required codes; no auth)
3. MCP create_task  — or —  POST /v1/tasks              (form with ≥1 input field)
4. MCP wait_for_task — or — GET  /v1/tasks/{id}/wait
5. Read response.form_data
```

Unclaimed orgs: 3 free **public** tasks only. Guild/direct routing needs a claimed org (`invite_human` / `POST /v1/org/invite`). Paid tasks need a funded wallet **and** a human-set spending limit > $0. There is no staffed SLA — humans may or may not claim public tasks.

See [`skills/source/SKILL.md`](./skills/source/SKILL.md) for the callable tools, then fetch the live canonical skill for forms, guilds, billing, and the full API.

## Dual listing

| Plugin | Call | Means | Does not mean |
|--------|------|-------|----------------|
| **SanctifAI Source — Hire a Human** (this package) | MCP `/mcp` + REST `/v1` | Ask registered marketplace workers; structured answers back | Staffed enterprise hire-desk; SLA unless contracted |
| **SanctifAI Trust — Proof of Human** | Chat bridge attestations | Participation seal (WebAuthn + attested payload) | Legal identity; “answer was correct”; staffed review desk |

## License

Apache License 2.0 — see the repo root [`LICENSE`](../../LICENSE) and [`NOTICE`](../../NOTICE). Apache-2.0 covers this skills/plugin packaging only; it does not grant trademark rights, does not grant patents beyond Apache-2.0 §3, and does not license the Source SaaS/service.

## Version

Plugin package: **1.0.0** (first public Source home in `sanctifai/skills`).
Bundled skill is a packaging wrapper; live SoT remains [https://app.sanctifai.com/agents/source/skill.md](https://app.sanctifai.com/agents/source/skill.md).
