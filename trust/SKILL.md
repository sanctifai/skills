---
name: sanctifai-trust-proof-of-human
description: Integrate SanctifAI Trust Proof-of-Human attestations. Use when an app needs cryptographic proof a human performed a task or human-in-the-loop verification.
homepage: https://trust.sanctifai.com
version: 2.0.0
updated: 2026-09-07
---

# SanctifAI Trust — Proof of Human

SanctifAI Trust turns a unit of human work into a verifiable **Proof of Human**:
the person confirms presence with WebAuthn (Touch ID / Windows Hello / passkey),
and you get a privacy-preserving participation plus a public certificate URL
(and optional on-chain seal). Raw task data never leaves the client — only
`0x`+SHA-256 commitments are sent.

Use this when you need to **prove a human did something** (approved a wire,
reviewed content, signed off a release, completed a gig) and receive
`participation_id` + `certificate_url`.

**Base API:** `https://trust.sanctifai.com`. Three equal surfaces, same outcome:

1. **Embedded** — Proof of Human inside the customer's own app (`/api/presence/*`, API key on their backend). No package install.
2. **Chrome extension** — SanctifAI Chrome extension + hosted `sanctifai-presence.js` (worker-bound identity). No package install.
3. **Chat bridge** — attestations minted for an AI agent via the hosted bridge at `https://bridge.trust.sanctifai.com`.

Local Trust on `localhost` is the docs site, not the production API.

## Choose a surface

Pick the surface that matches where the human actually works. All three mint the
same participation + `certificate_url`.

```
                         Where does the human work?
                                    │
         ┌──────────────────────────┼──────────────────────────┐
         │                          │                          │
         ▼                          ▼                          ▼
    You control the           External / BYO             AI agent / chat
    app (employees,           workforce, or an           with no WebAuthn
    your product UI)          app you don't control      browser context
         │                          │                          │
         ▼                          ▼                          ▼
     EMBEDDED                   EXTENSION                  CHAT BRIDGE
     REST /api/presence/*       Chrome extension +         Hosted mint →
     API key on YOUR backend    sanctifai-presence.js      approve_url in Chrome
     You supply user_id         Worker supplies identity   → poll → certificate_url
```

- **Embedded** — default for apps you control. WebAuthn runs in your page.
- **Extension** — the worker installs the SanctifAI extension and attests from pages you don't own.
- **Chat bridge** — the agent has no `navigator.credentials`; a human opens the approval link in Chrome.

Don't pick Extension merely because credentials are missing — get Embedded
credentials instead.

## Credentials

**Embedded** needs two values on the **backend only**:

```bash
TRUST_TENANT_ID=your_tenant_id   # ≤ 12 chars
TRUST_API_KEY=sk_live_...        # never in the browser, never in agent chat
```

Sign up at **https://trust.sanctifai.com**. The **Developer** plan is free
forever (no trial, no expiry): **100 attestations/month** (calendar month, UTC),
**1 tenant**, **unlimited reviewers**, 30-day audit history. Onboarding
provisions the tenant and a default API key (`sk_live_…`, shown once). Paid
plans raise limits — see [pricing](https://sanctifai.com/trust/pricing).

**Extension:** the worker configures tenant / user / key / RP ID in the
extension. Your page never holds those secrets.

**Chat bridge:** the agent never holds the API key. The hosted bridge does.

Register the Embedded origin + RP ID (`hostname` only, e.g. `app.example.com`)
on the tenant allowlist, or presence calls are rejected. Console → Tenants →
Manage for Tenant ID, keys, and origins. Don't guess or fabricate IDs.

## Shared rules

These apply on every surface. Don't restate them per flow.

### Taxonomy

`task_type` and `domain` are **fixed 3-letter codes** (enums). Descriptive
strings are rejected. Put the human-readable title in `task_subtype` (≤ 200
chars) and the detail inside `taskData` / `resultData` **before** hashing.

- **`task_type`** (required): `ENT ANN COL RND EVA RPA ORC GEN MOD STT TRA DSN DEV CXO SLS CMP ANA PMT CUR`. Default `GEN`.
- **`domain`** (optional): `GEN AUT DFS EDU ENE FIN INS HRM HOS LOG TRN LEG MED MDA RTL ROB SPT TEC GOV AGR REA TEL ESG`. Default `GEN`.

```txt
# WRONG
task_type: "PHARMABOT_TREATMENT_REVIEW"

# RIGHT
task_type:    "EVA"
domain:       "MED"
task_subtype: "Treatment plan review"
```

Full labels: [reference.md](reference.md).

### Privacy

The **certificate URL is public**. No PII in `user_id`, `task_subtype`,
`task_id`, `taskData`, or `resultData`. Use opaque IDs (`emp_a8f3c2`, not
`jane.doe@company.com`). Keep human-readable detail in your own system.

### After-the-fact proof

Trust proves a human participated and binds SHA-256 commitments. It does **not**
store raw `taskData` / `resultData`. To prove later *what* was attested:

1. Retain the exact payload that was hashed.
2. Re-hash it the same way: key-sorted JSON, then SHA-256, then `0x` + hex.
3. Match `task_commitment` / `result_commitment` on the certificate.

```js
JSON.stringify(payload, Object.keys(payload).sort())  // then SHA-256 → 0x+hex
```

A different key order or encoding will not match. Discard the payload and you
still have proof a human showed up — not proof of contents. `task_subtype` is
only the public headline.

### Fields at a glance

You send opaque IDs, taxonomy codes, a short `task_subtype`, and commitments.
Success returns `participation_id`, `certificate_url`, and usually `qr_url`.
Embedded also requires `idempotency_key` (UUID, generated on the backend).
Full request/response and EAS V3 matrices: [reference.md](reference.md).

Never ship a Trust API key to client JavaScript. Embedded mints the session on
your backend; Extension keeps the key in the worker's extension; Chat bridge
holds it on the hosted service.

## Surface A — Embedded

WebAuthn runs in **your** page. Mint the presence session on **your** backend
(it holds the key) and pass only `session_id` to the browser.

**Minimum flow:** enroll the reviewer's passkey if they don't have one on this
device (`POST /api/webauthn/registration/options` + `/verify` — see
[reference.md](reference.md)), then start → options → `navigator.credentials.get()`
→ verify. If `presence/options` returns **404 No WebAuthn credentials found**,
enroll once and retry.

`registration/verify` needs `tenant_id`, `user_id`, `rp_id`, `challenge_id`,
and `credential` including `clientExtensionResults`.

**Backend** (holds `TRUST_API_KEY`):

```js
// POST /api/trust/presence/start  — your server
export async function POST(req) {
  const body = await req.json();
  const userId = await getEmployeeIdFromSession(req); // from YOUR session, not the body
  const r = await fetch('https://trust.sanctifai.com/api/presence/start', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${process.env.TRUST_API_KEY}` },
    body: JSON.stringify({
      ...body,
      tenant_id: process.env.TRUST_TENANT_ID,
      user_id: userId,
      idempotency_key: crypto.randomUUID(),
    }),
  });
  const { session_id } = await r.json();
  return Response.json({ session_id });
}
```

**Browser** (raw task data stays local):

```js
const b64uToBuf = (s) => { const p = s.replace(/-/g,'+').replace(/_/g,'/').padEnd(s.length+(4-s.length%4)%4,'='); const b = atob(p); const u = new Uint8Array(b.length); for (let i=0;i<b.length;i++) u[i]=b.charCodeAt(i); return u.buffer; };
const bufToB64u = (buf) => { const u = new Uint8Array(buf); let s=''; for (let i=0;i<u.length;i++) s+=String.fromCharCode(u[i]); return btoa(s).replace(/\+/g,'-').replace(/\//g,'_').replace(/=+$/,''); };
const sha256Hex = async (d) => { const j = typeof d==='string'?d:JSON.stringify(d,Object.keys(d).sort()); const h = await crypto.subtle.digest('SHA-256', new TextEncoder().encode(j)); return '0x'+Array.from(new Uint8Array(h)).map(b=>b.toString(16).padStart(2,'0')).join(''); };

const API = 'https://trust.sanctifai.com';

async function createAttestation(taskData, resultData, { taskId, taskType='GEN', domain='GEN', taskSubtype } = {}) {
  const task_commitment = await sha256Hex(taskData);
  const result_commitment = await sha256Hex(resultData);

  const { session_id } = await fetch('/api/trust/presence/start', {
    method: 'POST', headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      task_id: taskId ?? `task-${Date.now()}`,
      task_type: taskType, domain, task_subtype: taskSubtype,
      task_commitment, result_commitment, bond_eligible: true,
      rp_id: location.hostname, origin: location.origin,
    }),
  }).then(r => r.json());

  const { options } = await fetch(`${API}/api/presence/options`, {
    method: 'POST', headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ session_id }),
  }).then(r => r.json());

  const cred = await navigator.credentials.get({ publicKey: {
    ...options,
    challenge: b64uToBuf(options.challenge),
    allowCredentials: (options.allowCredentials || []).map(c => ({ ...c, id: b64uToBuf(c.id) })),
  }});

  return fetch(`${API}/api/presence/verify`, {
    method: 'POST', headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ session_id, expedite: true, credential: {
      id: cred.id, rawId: bufToB64u(cred.rawId), type: cred.type,
      clientExtensionResults: cred.getClientExtensionResults?.() ?? {},
      response: {
        clientDataJSON: bufToB64u(cred.response.clientDataJSON),
        authenticatorData: bufToB64u(cred.response.authenticatorData),
        signature: bufToB64u(cred.response.signature),
        userHandle: cred.response.userHandle ? bufToB64u(cred.response.userHandle) : null,
      },
    }}),
  }).then(r => r.json()); // -> { participation_id, certificate_url, qr_url, ... }
}
```

Enrollment, Origin-forwarding for proxied enroll, and CORS: [reference.md](reference.md).

## Surface B — Extension

The worker installs the SanctifAI Chrome extension and configures **their**
tenant id, user id, API key, and RP id. Your page includes only the hosted
script — no secrets, **no** `user_id` from the page. Identity is worker-bound
and portable across the customers they work for.

```html
<script src="https://trust.sanctifai.com/sanctifai-presence.js"></script>
<script>
  await SanctifAIPresence.waitForReady(5000);
  const result = await SanctifAIPresence.createAttestation({
    taskData: { item: 'POST-913', content: '…' },
    resultData: { decision: 'approved' },
    taskType: 'GEN',
  });
  // result.participation_id, result.certificate_url, result.qr_url
</script>
```

`window.SanctifAIPresence` exposes `createAttestation`, `detectExtension`,
`isReady`, `waitForReady`, and `sha256Hex`.

## Surface C — Chat bridge

For agents with no WebAuthn context. Default
`APP_BASE_URL=https://bridge.trust.sanctifai.com`. Mint a request, the human
opens `approve_url` in Chrome, you poll until `certificate_url`. Do not invent
localhost approve links. Never request or print `TRUST_API_KEY`.

Self-host the plugin only if you need a custom allowlisted origin.

```
1. POST {APP_BASE_URL}/api/v1/attestations
2. Human opens approve_url in Chrome (passkey)
3. Poll GET {APP_BASE_URL}/api/v1/attestations/{id}
   or GET {APP_BASE_URL}/api/v1/attestations/{id}/wait
4. status=completed → certificate_url (hosted on trust.sanctifai.com)
```

**`POST {APP_BASE_URL}/api/v1/attestations`**

```json
{
  "task_type": "GEN",
  "domain": "GEN",
  "task_subtype": "Chat approval",
  "requesting_agent": "Support bot",
  "taskData": { "summary": "opaque task payload" },
  "resultData": { "decision": "approved" }
}
```

Optional `requesting_agent` (≤ 120 chars) is shown on the approve page.
Empty/whitespace values are omitted.

**Response:** `{ attestation_id, approve_url, status: "pending" }`

Tell the human: *Open this HTTPS link in Chrome and confirm with your device
passkey: `{approve_url}`*. If they have no passkey on this origin, the page
enrolls and retries.

**`GET …/wait`** blocks up to ~25s for a terminal status; if still `pending`,
call it again.

**Done when:** `{ status: "completed", certificate_url: "https://trust.sanctifai.com/certificate/..." }`

Retain the exact `taskData` / `resultData` you posted (see Shared rules). The
bridge approve client uses the same `sha256Hex` helper as Embedded.

Shorter chat-only skill: https://bridge.trust.sanctifai.com/skill.md

## Verify

Success is a non-empty `participation_id` + `certificate_url` (Chat bridge:
`status=completed` + `certificate_url`). Open the cert. Show `qr_url` if you
have a UI.

## Deeper material

- [reference.md](reference.md) — REST endpoints, taxonomy labels, enrollment, CORS, errors
- Product: https://trust.sanctifai.com
- Chat bridge skill: https://bridge.trust.sanctifai.com/skill.md
