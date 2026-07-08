---
name: sanctifai-trust-proof-of-human
description: Integrate SanctifAI Trust Proof-of-Human attestations. Use when an app needs cryptographic proof a human performed a task or human-in-the-loop verification.
homepage: https://trust.sanctifai.com
---

# SanctifAI Trust — Proof of Human

SanctifAI Trust turns a unit of human work into a verifiable **Proof of Human**
attestation: a person confirms presence with WebAuthn (Touch ID / Windows Hello /
passkey), and the platform records a privacy-preserving participation plus an
optional on-chain seal and a public certificate URL + QR code. Raw task data
never leaves the browser — only `0x`+SHA-256 commitments are sent.

Use this skill when a user wants to **prove a human did something** (approved a
wire, reviewed content, signed off a release, completed a gig task) and get a
`participation_id` + `certificate_url` back.

**No package install required.** You integrate by calling the REST API directly
(embedded path) or by including one hosted `<script>` (extension path). Base API:
`https://trust.sanctifai.com`.

## Step 0 — Use the production API (not a local Trust instance)

Set `TRUST_API_BASE_URL=https://trust.sanctifai.com` and call that. **A Trust
site running on `http://localhost:3000` is the docs/demo site — treat it as
documentation, not the API target**, unless you have explicitly confirmed it runs
the full API with your tenant's origin allowlist. Pointing integration at a local
Trust instance is a common one-shot failure: it causes origin-allowlist and CORS
errors that look like API bugs. Unless told otherwise, target production.

## Step 1 — Get tenant credentials first (don't skip to the extension)

Embedded integration needs two tenant credentials that live **only on your
backend**:

```bash
TRUST_TENANT_ID=your_tenant_id   # ≤ 12 chars
TRUST_API_KEY=sk_live_...        # tenant-scoped secret — NEVER in client code
```

### If you don't have a Tenant ID or API key yet

Sign up at **https://trust.sanctifai.com**. New accounts go through an onboarding
wizard that starts a **free 7-day trial** and provisions what you need:

- **1 tenant** (with Tenant ID)
- **1 default API key** (`sk_live_…`) — copy it during setup; it is shown once
- Plan limits on the trial: **1 tenant**, **10 users**, **100 attestations**

After signup, open **Console → Tenants → Manage** to create additional API keys or
copy your Tenant ID. Store both values in your backend environment (`.env` /
secret manager), not in client code or agent chat logs you don't trust.

If the integrator already has credentials, write them into the backend at build
time. If they don't, **direct them to trust.sanctifai.com** — do not guess or
fabricate IDs.

> **Do not fall back to the Extension path just because you lack an API key.** A
> missing key is not a reason to assume the worker will supply one. Obtain the
> credentials via the Trust console (or ask the user for values they received
> there) and build **embedded** — it is the default. Whatever origin runs the
> attestation must be on the tenant's allowlist (see Hard rules / reference.md).

## Step 2 — Pick the integration path (only if embedded is genuinely impossible)

Ask exactly one question: **does the integrator control the app where the human
works?**

```
        Does the integrator CONTROL the app where the human works?
                                 │
        ┌────────────────────────┴────────────────────────┐
       YES — their own app / employees          NO — a 3rd-party app they don't
        │                                         control, or external/BYO workforce
        ▼                                              ▼
   EMBEDDED (app-bound) ← DEFAULT                 EXTENSION (worker-bound)
   • WebAuthn runs in THEIR page                 • WebAuthn runs in the worker's
   • API key stays on THEIR backend                SanctifAI Chrome extension
   • THEY supply user_id                          • the WORKER supplies identity
   • REST API, no extension                       • one hosted <script>, no API key
   (REST: /api/presence/*)                        (script: sanctifai-presence.js)
```

Both paths produce the same participation + certificate. **Default to embedded.**
Choose Extension *only* when the humans are genuinely external / bring-your-own
workforce, or the app is not controlled by the integrator — never merely because
a credential is missing.

## Taxonomy codes (hard rule — wrong values fail)

`task_type` and `domain` are **fixed 3-letter taxonomy codes**, validated as
enums. Descriptive strings are rejected.

- **`task_type`** (required): one of
  `ENT ANN COL RND EVA RPA ORC GEN MOD STT TRA DSN DEV CXO SLS CMP ANA PMT CUR`.
  Default `GEN`.
- **`domain`** (optional): one of
  `GEN AUT DFS EDU ENE FIN INS HRM HOS LOG TRN LEG MED MDA RTL ROB SPT TEC GOV AGR REA TEL ESG`.
  Default `GEN`.
- **`task_subtype`** (optional): free-text label, **max 200 chars** — this is
  where a human-readable description goes (it renders as the certificate title).

```txt
# WRONG — descriptive label in a code field → rejected
task_type: "PHARMABOT_TREATMENT_REVIEW"

# RIGHT — codes in the code fields, label in task_subtype, detail in taskData
task_type:    "EVA"
domain:       "MED"
task_subtype: "Treatment plan review"
# full details go inside taskData/resultData BEFORE hashing
```

The full code lists with labels live in `reference.md`.

## What's in an attestation record (and why you pass each field)

Trust splits data across three layers: **what you send at integration time**,
**what SanctifAI stores** (participation database + public certificate), and
**what is anchored on-chain** (Ethereum Attestation Service on Base). Raw
`taskData` / `resultData` never leave the browser — only salted SHA-256
commitments and selected metadata.

### Fields you supply (`POST /api/presence/start` — embedded path)

| Field | Public certificate | On-chain (EAS V3) | Trust DB | Why you pass it |
|---|---|---|---|---|
| `tenant_id` | Yes | No — **tenant commitment** only | Yes | Scopes the proof to your Trust tenant |
| `user_id` | Yes | No | Yes | Binds the human reviewer — use an **opaque internal ID**, not PII |
| `task_id` | Yes | No — **task commitment** only | Yes | Stable id for the unit of work / idempotency |
| `task_type` | Yes (label) | No | Yes | 3-letter taxonomy — category of human work |
| `domain` | Yes (label) | No | Yes | 3-letter industry/domain code |
| `task_subtype` | Yes (headline) | No | Yes | Short certificate title — **no PII** |
| `task_commitment` | Yes (hash) | Yes (`taskCommitment`) | Yes | Proves task payload without revealing it |
| `result_commitment` | Yes (hash) | Yes (`resultCommitment`) | Yes | Proves outcome/decision without revealing it |
| `human_fp` | Yes | Yes (`humanFingerprint`) | Yes | Pseudonymous WebAuthn / passkey fingerprint |
| `bond_eligible` | Yes | Yes | Yes | Whether attestation may back a bond product |
| `rp_id` / `origin` | RP ID on cert | No | Yes | Domain binding and allowlist enforcement |
| `taskData` / `resultData` | **No** (hashed only) | **No** | **No** raw copy | Hashed client-side; plaintext never sent |

Optional `customer_id` (UUID) may appear on the certificate if supplied — still
must not contain or encode PII.

### On-chain EAS attestation (V3 — public on Base)

| On-chain field | Comes from | Purpose |
|---|---|---|
| `taskCommitment` | Client SHA-256 of task payload | Tamper-evident task binding without disclosure |
| `humanFingerprint` | WebAuthn enrollment | Proves the same human authenticator signed |
| `tenantCommitment` | Salted hash of `tenant_id` | Links proof to tenant without publishing the slug |
| `resultCommitment` | Client SHA-256 of result payload | Tamper-evident outcome binding |
| `ts` | Anchor time | When the proof was sealed |
| `bondEligible` | Request flag | Bond-eligibility marker |

Plain `tenant_id` and `task_id` strings are **not** written to chain in V3 — only
their commitments (plus `humanFingerprint` and timestamps).

### After `presence/verify` — what the integrator gets back

| Field | Purpose |
|---|---|
| `participation_id` | Primary ID — certificate URL, QR, and API lookups |
| `certificate_url` | Public proof page anyone can open and verify |
| `qr_url` | QR seal image for your UI |
| `verification_url` | EAS Scan / block explorer link when anchored |
| `human_fp` | Reviewer's pseudonymous fingerprint (same as enrolled passkey) |

### Privacy rule of thumb

Treat the **certificate URL as public**. If you would not publish a value there,
do not put it in `user_id`, `task_subtype`, `task_id`, or inside
`taskData`/`resultData` (hashes are public; metadata above is shown on the cert).
Store human-readable detail in your own systems; keep Trust payloads pseudonymous.

## Step 3a — Embedded (app-bound): call the REST API directly

The Trust API key must **never** reach the browser. Mint the presence session on
the integrator's backend (it holds the key) and pass only the `session_id` to the
page. `presence/options` and `presence/verify` are authorized by `session_id`, so
the browser never needs the key.

### Embedded minimum flow (don't ship attestation-only)

A complete embedded integration **enrolls the reviewer, then attests**:

```txt
1. Register the reviewer's passkey if they don't have one on this device
   (POST /api/webauthn/registration/options + /verify — see reference.md).
2. Start the presence session (backend; holds the API key).
3. Fetch presence options.
4. Prompt navigator.credentials.get().
5. Verify the credential → participation.
6. Display participation_id, certificate_url, and the qr_url QR image.
```

Skipping step 1 is the most common one-shot failure: `presence/verify` returns
`No WebAuthn credentials found` until the reviewer has enrolled a passkey.

**Backend** (server-side; holds `TRUST_API_KEY`):

```js
// POST /api/trust/presence/start  — your server
export async function POST(req) {
  const body = await req.json();                 // commitments + task fields from the browser
  const userId = await getEmployeeIdFromSession(req); // authorize from YOUR session, not the body
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

**Browser** (runs WebAuthn; raw task data stays local):

```js
// tiny helpers (WebAuthn needs base64url <-> ArrayBuffer)
const b64uToBuf = (s) => { const p = s.replace(/-/g,'+').replace(/_/g,'/').padEnd(s.length+(4-s.length%4)%4,'='); const b = atob(p); const u = new Uint8Array(b.length); for (let i=0;i<b.length;i++) u[i]=b.charCodeAt(i); return u.buffer; };
const bufToB64u = (buf) => { const u = new Uint8Array(buf); let s=''; for (let i=0;i<u.length;i++) s+=String.fromCharCode(u[i]); return btoa(s).replace(/\+/g,'-').replace(/\//g,'_').replace(/=+$/,''); };
const sha256Hex = async (d) => { const j = typeof d==='string'?d:JSON.stringify(d,Object.keys(d).sort()); const h = await crypto.subtle.digest('SHA-256', new TextEncoder().encode(j)); return '0x'+Array.from(new Uint8Array(h)).map(b=>b.toString(16).padStart(2,'0')).join(''); };

const API = 'https://trust.sanctifai.com';

async function createAttestation(taskData, resultData, { taskId, taskType='GEN', domain='GEN', taskSubtype } = {}) {
  // 1. hash inputs locally — raw data never leaves the page
  const task_commitment = await sha256Hex(taskData);
  const result_commitment = await sha256Hex(resultData);

  // 2. mint the session via YOUR backend (keeps the API key server-side)
  const { session_id } = await fetch('/api/trust/presence/start', {
    method: 'POST', headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      task_id: taskId ?? `task-${Date.now()}`,
      task_type: taskType, domain, task_subtype: taskSubtype, // codes only; label in task_subtype
      task_commitment, result_commitment, bond_eligible: true,
      rp_id: location.hostname, origin: location.origin,
    }),
  }).then(r => r.json());

  // 3. fetch WebAuthn options
  const { options } = await fetch(`${API}/api/presence/options`, {
    method: 'POST', headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ session_id }),
  }).then(r => r.json());

  // 4. prompt the platform authenticator
  const cred = await navigator.credentials.get({ publicKey: {
    ...options,
    challenge: b64uToBuf(options.challenge),
    allowCredentials: (options.allowCredentials || []).map(c => ({ ...c, id: b64uToBuf(c.id) })),
  }});

  // 5. verify + create the participation
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

Each reviewer must enroll a passkey **once per device** before attesting — see
[reference.md](reference.md) (enrollment, the `Origin`-forwarding rule for
proxied enrollment, and CORS/proxy notes).

## Step 3b — Extension (worker-bound): one hosted script

The worker installs the SanctifAI Chrome extension and configures their **own**
tenant id, user id, API key, and RP id. Your page includes only the hosted bridge
script — it never holds per-worker secrets and supplies **no** `user_id`.

```html
<script src="https://trust.sanctifai.com/sanctifai-presence.js"></script>
<script>
  await SanctifAIPresence.waitForReady(5000);
  const result = await SanctifAIPresence.createAttestation({
    taskData: { item: 'POST-913', content: '…' }, // hashed client-side
    resultData: { decision: 'approved' },          // hashed client-side
    taskType: 'GEN', // optional, taxonomy code
  });
  // result.participation_id, result.certificate_url, result.qr_url
</script>
```

The script exposes `createAttestation`, `detectExtension`, `isReady`,
`waitForReady`, and `sha256Hex` on `window.SanctifAIPresence`. Identity comes from
the worker's extension config, so the attestation binds to the **worker** and is
portable across the customers they work for.

## Hard rules

1. **Default to embedded; obtain credentials, don't dodge to the extension.**
   Embedded needs `TRUST_TENANT_ID` + `TRUST_API_KEY` on the backend — have them
   or prompt the user. A missing key is never a reason to pick the extension.
2. **Target production** `https://trust.sanctifai.com`. A local Trust site is docs
   only unless confirmed to run the full API with your tenant allowlist.
3. **Never ship a Trust API key to client JavaScript.** Embedded → mint the
   session on the backend. Extension → the key lives in the worker's extension.
4. **`task_type` / `domain` are fixed 3-letter codes** (see Taxonomy codes).
   Descriptive labels go in `task_subtype` (≤ 200 chars) or inside
   `taskData`/`resultData` before hashing — never in a code field.
5. **`taskData` / `resultData` stay raw on the client** — they are hashed into
   `0x`+SHA-256 commitments; only hashes are transmitted.
6. **No PII on the public certificate.** Fields that can appear on the certificate
   or proof URL — especially `task_subtype`, labels inside `taskData`/`resultData`,
   and any display metadata — must **not** contain personally identifiable
   information (names, emails, phone numbers, account numbers, government IDs,
   full addresses, etc.). Use opaque internal IDs for `user_id` (e.g.
   `emp_a8f3c2`, not `jane.doe@company.com`). The Tenant ID is already scoped to
   the integrator; do not embed customer PII in tenant-facing attestation payloads.
   Put human-readable detail in your own app/database; keep Trust payloads
   pseudonymous.
7. **Register production origin(s) + RP ID** with the tenant, or the API's
   allowlist rejects the domain.
8. **`rp_id` is the hostname only** (e.g. `app.example.com`, or `localhost` in dev).
9. **The attestation step runs in a browser** (it calls `navigator.credentials`).

## Before you ask the user to test (smoke test)

Sanity-check the wiring **without** completing a real attestation:

```txt
- POST /api/webauthn/registration/options  → returns challenge_id + options
- POST /api/presence/start                  → returns session_id
- POST /api/presence/options                → returns options.challenge
```

If any of these returns HTML, a 401/403, or a CORS error, fix the base URL, API
key, or origin allowlist **before** going further — don't prompt the user yet.

## UI acceptance criteria

A complete UI shows, after a successful attestation:

```txt
- participation_id
- certificate_url  (clickable link — open it to confirm the public proof renders)
- qr_url           (rendered as a QR image, not just a link)
- verification_url (if present — the on-chain explorer link)
- clear error guidance when the reviewer has not enrolled a passkey
```

## Verify it worked

A successful call returns JSON with a non-empty `participation_id` and a
`certificate_url`. Open the `certificate_url` to confirm the public proof renders.

## Deeper material

- [reference.md](reference.md) — REST endpoints, full taxonomy code lists,
  obtaining credentials, one-time passkey enrollment, the `Origin`-forwarding rule
  for proxied enrollment, error/troubleshooting tables, CORS/same-origin-proxy
  guidance, response shapes.
- Docs: https://trust.sanctifai.com
