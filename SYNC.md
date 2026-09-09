# Skill sync

This repo is a **read-mostly mirror** for marketplace crawlers. Agents fetching HTTP should use the live product URLs.

Do **not** hand-edit `source/SKILL.md` or `trust/SKILL.md`. Keep those folder names and the uppercase `SKILL.md` filenames — Cursor/plugin scanners need them.

## Canonical map

| Product | Web (live SoT) | This repo | SoT file (writer) |
|---------|----------------|-----------|--------------------|
| Source | https://app.sanctifai.com/agents/source/skill.md | `source/SKILL.md` | `docs/skill.md` in app.sanctifai.com |
| Trust (full website skill) | https://trust.sanctifai.com/agents/trust/skill.md | `trust/SKILL.md` | `apps/web/public/agents/trust/skill.md` in sanctifai-trust |
| Trust chat-bridge plugin | (bundled) | `plugins/sanctifai-trust/skills/proof-of-human/SKILL.md` | this repo |

Compatibility aliases still exist on the apps (`/agents/skill.md`, `/agent-skill.md`, `/skill.md` on the product hosts). Prefer `/agents/{product}/skill.md`.

## Sync matrix

| Artifact | Who writes | On which push | Lands here |
|---------|------------|---------------|------------|
| Source product skill | `docs/skill.md` in the **app.sanctifai.com** repo | `mirror-skill.yml` on production change | `source/SKILL.md` |
| Trust product skill | `apps/web/public/agents/trust/skill.md` in **sanctifai-trust** | `mirror-skill.yml` on production change | `trust/SKILL.md` |
| Trust Chat-bridge plugin | this repo (`plugins/sanctifai-trust/`) | PRs merged to this repo | `plugins/sanctifai-trust/` |

Hand-edits to the mirrored product skills are overwritten on the next sync. Change the product-repo file, not the copy here.
