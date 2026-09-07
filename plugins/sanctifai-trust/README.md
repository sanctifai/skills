# SanctifAI Trust

Installable **Cursor / Claude Code / Codex / Agent** plugin for **Chat bridge Proof of Human**.

The agent cannot run WebAuthn in-card. This plugin teaches it to mint an attestation on the hosted Chat bridge, send the human an `approve_url` (Chrome + passkey), and poll until `certificate_url`.

**Homepage:** [https://trust.sanctifai.com](https://trust.sanctifai.com)
**Default `APP_BASE_URL`:** `https://bridge.trust.sanctifai.com`
**Live skill:** [https://bridge.trust.sanctifai.com/skill.md](https://bridge.trust.sanctifai.com/skill.md)

The Chat bridge **runtime** stays in the private `sanctifai/sanctifai-trust` monorepo (`apps/chat-bridge`). This directory is **packaging only** — public so the plugin can later be submitted to the Cursor marketplace. Do not point agents at legacy hosts such as `trust-agent-c94n`.

This plugin is **not** the product-wide Trust skill (`trust/SKILL.md` in this repo), which covers Embedded + Extension + Chat bridge. Use that file when integrating Trust into an app you control. Use **this** plugin when the human is in a chat / agent session with no browser WebAuthn context.

## Layout

```
plugins/sanctifai-trust/
├── plugin.json                 # Agent Plugins 1.0.0
├── .cursor-plugin/plugin.json  # Cursor
├── .claude-plugin/plugin.json  # Claude Code
├── .codex-plugin/plugin.json   # Codex / ChatGPT
├── assets/logo.svg
├── README.md
└── skills/proof-of-human/SKILL.md
```

## Install

Point the harness at this plugin folder (`plugins/sanctifai-trust` in [sanctifai/skills](https://github.com/sanctifai/skills)), or add this repository as a Cursor marketplace (see `.cursor-plugin/marketplace.json` at the repo root).

Cursor team marketplace: Dashboard → Plugins → import `https://github.com/sanctifai/skills`. Cursor reads `.cursor-plugin/marketplace.json` and lists `sanctifai-trust`.

Local Cursor test: copy this folder to `~/.cursor/plugins/local/sanctifai-trust`.

Claude Code / Codex: install from this plugin directory (each harness reads its own `.*-plugin/plugin.json`). Skills resolve from `./skills`.

This package is **not** on the public Cursor Marketplace yet. Marketplace submit is a follow-up; do not treat this README as a published listing.

## Usage

Default every URL off `APP_BASE_URL=https://bridge.trust.sanctifai.com`:

1. `POST {APP_BASE_URL}/api/v1/attestations`
2. Give the human the returned `approve_url` (Chrome, HTTPS, passkey)
3. Poll `GET {APP_BASE_URL}/api/v1/attestations/{id}` (or `/wait`)
4. When `status` is `completed`, paste `certificate_url`

The agent never holds `TRUST_API_KEY`. The hosted bridge does. Self-host the bridge only if you need a custom allowlisted origin.

See [`skills/proof-of-human/SKILL.md`](./skills/proof-of-human/SKILL.md) for the full flow, hashing rules, and hard constraints.

## Version

Plugin package: **1.0.0** (first public home in `sanctifai/skills`).
Bundled skill frontmatter version follows the live Chat bridge skill (`2.0.0` at copy time).
