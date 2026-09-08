---
name: proof-of-human
description: Use this whenever an agent must prove a qualified human participated — bind points, approvals, regulated actions, or sending work outside the session. Obtains cryptographic Proof of Human (WebAuthn-backed attestation + portable certificate_url) on demand. When WebAuthn cannot run in-card, mint via this Chat bridge (POST attestation → human opens Chrome approve_url → poll until certificate_url).
homepage: https://trust.sanctifai.com
version: 2.0.0
updated: 2026-09-07
---

# Proof of Human

SanctifAI Trust Chat bridge — one of three Trust surfaces (Embedded, Chrome
extension, Chat bridge). All three produce the same participation +
`certificate_url`. This skill is the chat path: you cannot run WebAuthn
in-card, so you mint here, a human opens Chrome, and you poll for the cert.

**Default `APP_BASE_URL`:** `https://bridge.trust.sanctifai.com`

Always use the `approve_url` the API returns. Never invent `localhost` links.
The agent never holds `TRUST_API_KEY` — the hosted bridge does. Self-host this
plugin only if you need a custom allowlisted origin.

Embedded and Extension: https://trust.sanctifai.com/agent-skill.md

## Flow

```
1. POST {APP_BASE_URL}/api/v1/attestations
2. Give the human approve_url — they open it in Chrome
3. Poll GET {APP_BASE_URL}/api/v1/attestations/{id}
   or GET {APP_BASE_URL}/api/v1/attestations/{id}/wait
4. When status is completed, paste certificate_url
```

## 1. Create

`POST {APP_BASE_URL}/api/v1/attestations`

```json
{
  "task_type": "GEN",
  "domain": "GEN",
  "task_subtype": "Chat approval",
  "requesting_agent": "Sebastian - Dev Bot",
  "taskData": { "summary": "opaque task payload" },
  "resultData": { "decision": "approved" }
}
```

`task_type` and `domain` are 3-letter taxonomy codes. Put a short title in
`task_subtype` (max 200 chars). Optional `requesting_agent` is a short display
string (max 120 chars) shown on the approve page so the human can see which
agent asked for Proof of Human. Empty or whitespace-only values are omitted.
Do not put PII in any field — the certificate URL is public.

Response:

```json
{
  "attestation_id": "uuid",
  "approve_url": "https://bridge.trust.sanctifai.com/approve?token=...",
  "status": "pending"
}
```

## 2. Human step (Chrome)

Tell the human:

> Open this HTTPS link in Chrome and confirm with your device passkey:
> `{approve_url}`

The page talks to `https://trust.sanctifai.com` through this plugin
(`presence/start`, `presence/options`, `presence/verify`). If Trust returns
404 / "no credentials", the page enrolls a passkey on this origin and retries.

## 3. Poll

`GET {APP_BASE_URL}/api/v1/attestations/{attestation_id}`

`GET {APP_BASE_URL}/api/v1/attestations/{attestation_id}/wait`
waits up to ~25s for a terminal status. If still `pending`, call it again.

Done when:

```json
{
  "attestation_id": "uuid",
  "status": "completed",
  "certificate_url": "https://trust.sanctifai.com/certificate/..."
}
```

Paste `certificate_url`. That is the proof of **human presence**, not a
self-describing copy of the task.

## After-the-fact proof

Trust proves a human participated and binds SHA-256 commitments. It does
**not** store or republish raw `taskData` / `resultData` contents.

The **only** way to prove later what was in the participation is to
**retain the exact payload** hashed at attest time, present that same
data, **re-hash** it with the canonicalization below, and **match**
`task_commitment` / `result_commitment` on the certificate (and on-chain
`taskCommitment` / `resultCommitment` if a seal was used).

If you discard the payload, you still have proof a human showed up — not
proof of contents or context.

`task_subtype` is only a short public headline (certificate title, max 200
chars). It is not the full record.

The agent / integrator must keep the system-of-record copy. This bridge's
local store is ephemeral (lost on deploy/restart) and is not a content
archive. Trust makes your copy tamper-evident; it does not make the
certificate self-describing.

### Canonicalization this bridge actually uses

Commitments sent to Trust are computed **in the browser** when the human
clicks Approve (`app/approve/approve-client.tsx` calls `sha256Hex` from
`app/approve/webauthn.ts`). The server does **not** re-hash:
`POST /api/presence/start` forwards the client `task_commitment` /
`result_commitment` to Trust; `POST /api/presence/verify` only forwards
the WebAuthn assertion.

This app has **no** `lib/tokens.ts` and does **not** use a recursive
`stableStringify` on the approve path. Do not re-hash with a deep
key-sort helper — that will false-mismatch.

`sha256Hex(data)` is:

```js
async function sha256Hex(data) {
  const json =
    typeof data === 'string'
      ? data
      : JSON.stringify(data, Object.keys(data).sort());
  const digest = await crypto.subtle.digest(
    'SHA-256',
    new TextEncoder().encode(json)
  );
  return (
    '0x' +
    Array.from(new Uint8Array(digest))
      .map((b) => b.toString(16).padStart(2, '0'))
      .join('')
  );
}
```

- Strings are hashed as UTF-8 bytes (no JSON wrapping).
- Objects use `JSON.stringify(data, Object.keys(data).sort())` — sorted
  **top-level** keys only, **not** a deep recursive canonicalizer.
- The result is SHA-256, then `0x` + lowercase hex.

Because a string-array `JSON.stringify` replacer is applied at every
nesting level, nested properties whose names are **not** also top-level
keys of the root object are omitted from the hashed JSON. A recursive
key-sort (`stableStringify`) will not match. Copy this helper to
re-hash; do not invent a third algorithm.

Prefer flat `taskData` / `resultData` objects so nested-key dropping is
not a surprise.

## Grok Bot / Auto-review

Mint and poll are ordinary HTTPS calls to this bridge. On Grok Bot (and
similar clients), **Auto-review may block the first outbound POST/GET** to
`APP_BASE_URL` even though that host is the intended Trust chat bridge.

When that happens:

1. Tell the user Auto-review blocked the Trust mint (or poll).
2. Retry the **exact same** command with the client's approval-retry /
   "raise approval card" flag so they get Allow once / Deny.
3. Do **not** switch hosts, invent a workaround, or skip Proof of Human.

After they allow once, continue the normal flow (send `approve_url`, poll,
paste `certificate_url`).

## Hard rules

- Use `APP_BASE_URL` for every URL. Do not invent localhost approve links.
- Never request or print `TRUST_API_KEY`.
- Keep payloads pseudonymous. The certificate is public.
- Retain the exact `taskData` / `resultData` you posted. Trust will not
  give them back.
