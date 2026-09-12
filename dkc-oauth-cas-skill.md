---
name: dkc-oauth-cas
description: >-
  Integrate an app with DKC's CAS (Central Approval Service / "DKC OAuth") — the
  internal approval API at oauth.dhammakaya.network that routes an approval
  request to a supervisor (หัวหน้า) over email, LINE, and plain-text channels and
  then calls the app back with the decision. Use this whenever the user is
  building or changing an app (Laravel, Next.js, or any backend) that needs a
  supervisor to approve/reject something, or when they mention CAS, DKC OAuth,
  startApprove, RequestStatus, ApproveType, HeadKong / HeadSamnak, ExternalRefID,
  an approval CallbackURL, or wiring up an approve/reject callback — even if they
  don't name the skill. Also use when they ask to generate a CASService, an
  approval callback controller/route, or the CAS .env entries. Reach for this
  skill on Thai phrasings too, e.g. "ทำระบบขออนุมัติหัวหน้า", "ต่อ CAS", "ส่งขอ
  อนุมัติผ่านอีเมล/LINE".
---

# DKC CAS Approval Integration

CAS (Central Approval Service, internally called **DKC OAuth**) is DhammaKaya's
shared approval backend. An app hands CAS an approval request; CAS notifies the
right supervisor through email, LINE, and plain text; the supervisor approves or
rejects; CAS sends the decision to the app's callback URL. The app then
pulls the finalized status and updates its own records.

This skill exists because none of the CAS contract — the endpoints, the exact
payload fields, the `ApproveType` codes, the callback shape — is discoverable
from the outside. It's org-specific knowledge. The goal here is to generate
correct, consistent CAS wiring so a developer never has to reverse-engineer it
from an old controller again.

## The flow (what you're implementing)

1. **Request** — the app calls `startApprove(...)`, a POST to `CAS_REQUEST` with
   the requester, the app's own reference id, and how many approval levels are
   needed. CAS fans the request out to the supervisor over every channel.
2. **Wait** — the supervisor sees it in email / LINE / etc. and taps approve or
   reject.
3. **Callback** — **every time** a supervisor taps approve/reject, their browser
   is sent to the app's `CallbackURL` via **GET**, with `status` (`APPROVE` or
   `REJECT`) and `ExternalRefID` in the query string. For a 2-level approval this
   fires once per level.
4. **Sync** — on each callback the app calls `RequestStatus(...)`, a POST to
   `CAS_REQ_STATUS`, to read the authoritative overall `Status` and who acted,
   then updates its own record to match and shows a result page.

The app owns steps 1 and 4's business logic (which record to update, what the
result page says). CAS owns the notification and the human decision.

## Shared contract (framework-independent)

### .env

```env
CAS_REQUEST="https://oauth.dhammakaya.network/api/cas/request"
CAS_REQ_STATUS="https://oauth.dhammakaya.network/api/cas/reqstat"
CAS_CLIENT_ID=            # the app's CAS client id (from IT Dev)
CAS_CLIENT_SECRET=       # the app's CAS client secret (from IT Dev) — sent on startApprove
```

Both are POST endpoints that accept a JSON body and return JSON. Successful
responses carry the payload under a `data` key.

### startApprove request body

| Field | Req? | Meaning |
|---|---|---|
| `client_id` | required | The app's registered CAS client id (an integer). Each app has its own. **Do not hard-code a guessed value** — see "Getting the client credentials" below. |
| `client_secret` | required | The app's CAS client secret, paired with `client_id`, authenticating the app to CAS on the request. Keep it in env/config, never inline. |
| `ExternalRefID` | required | The app's own id/number for this task (e.g. a worksheet number like `WO022665`). CAS echoes this back on callback so the app can find its record. |
| `ApproveType` | required | How many approval levels: `1` = HeadKong only (หัวหน้ากอง), `2` = HeadKong **and** HeadSamnak (หัวหน้าสำนัก). |
| `Requester_ADUser` | one-of | AD username of the person asking for approval. Required **unless** `Requester_KongId` is given. |
| `Requester_KongId` | one-of | Division (กอง) id of the requester — the alternative to `Requester_ADUser`. Required **unless** `Requester_ADUser` is given. Supply **at least one** of the two. |
| `CallbackURL` | required | Absolute URL CAS sends the supervisor to via GET when they decide. Must be reachable from where they open the link (e.g. their browser). |
| `MsgSubject` | optional | Subject line of the approval request. |
| `MsgForHead` | optional | Plain-text message shown to the supervisor on **every** channel. |
| `MsgHTMLForHead` | optional | HTML message used for the **email** channel. |
| `MsgFlexForHead` | optional | LINE Flex **body content only** for the LINE channel — the single box component that goes *inside* `body`, **not** the full bubble and **not** wrapped in a `body:` key. CAS wraps it into the bubble itself. |

`MsgFlexForHead` is the value that would sit under a bubble's `body` — a box, on
its own:

```jsonc
// CORRECT — body content only (a box), no "body" key, no bubble wrapper
{
  "type": "box",
  "layout": "vertical",
  "contents": [
    { "type": "text", "text": "ขออนุมัติเปิดเน็ต WO022665", "weight": "bold" },
    { "type": "text", "text": "จาก: สมชาย", "size": "sm", "color": "#888888" }
  ]
}
// WRONG — do NOT send { "body": { ... } } or { "type": "bubble", "body": { ... } }
```

### startApprove response

On success the response `data` names the supervisor the request was routed to.
Use it to confirm to the requester who now has to approve:

| Field | Meaning |
|---|---|
| `success` | CAS accepted and routed the request (business-level success, distinct from the HTTP status) |
| `ADApprover` | AD username of the supervisor it was routed to |
| `RequestKey` | CAS's own key for this approval request — store it if you want to correlate later |
| `HeadFullName` | supervisor's full name |
| `Position` | supervisor's position |
| `Organization` | supervisor's organization / unit |
| `HeadShowEmail` | supervisor's email to display |

When both the HTTP call and `data.success` succeed, show the requester a
confirmation naming the approver — e.g. `ส่งให้หัวหน้าหน่วยงานอนุมัติแล้ว —
ผู้อนุมัติคือ {HeadFullName} ({Position}, {Organization}) อีเมล {HeadShowEmail}`.
Generate this confirmation in the `startApprove` usage path, not just a generic
"sent" message.

### RequestStatus request body

Send `client_id` and `ExternalRefID`. On success, `data` is the approval record
— the row with the **highest `ApproveStep`** (the latest step; for
`ApproveType = 2` this is the second-level record once it exists). Its `Status`
is the authoritative state to sync to. Fields:

| Field | Meaning |
|---|---|
| `client_id` | the app's CAS client id |
| `ExternalRefID` | the app's ref id |
| `ApproveType` | `1` or `2` |
| `ApproveStep` | which approval step this record represents |
| `Requester_ADUser` | who requested |
| `Approver_ADUser` | AD username of who approved this step |
| `Title` | request title |
| `Summary` | request summary |
| `DetailJSON` | app-supplied detail payload (JSON string) |
| `Status` | the record's approval status: `PENDING`, `APPROVE`, or `REJECT` |
| `CallbackURL` | the callback registered for this request |
| `created_at` / `updated_at` | timestamps |

### Callback contract

CAS calls `CallbackURL` with an HTTP **GET** — it's the link the supervisor
opens from the email / LINE message, so the app's callback renders a result page
in their browser. The decision arrives as **query-string** parameters: `status`
(`APPROVE` / `REJECT`) and `ExternalRefID`. **CAS calls it every time a
supervisor decides.** For `ApproveType = 2` (HeadKong + HeadSamnak) that means it
can be called more than once — once when the first head acts and again when the
second does. So a handler must **not** treat the first call as final.

The `status` query param is *that supervisor's* action; the request's
**authoritative overall state** comes from `RequestStatus.Status` — `PENDING`
while more levels remain, `APPROVE` / `REJECT` once it's settled. On each
callback, call `RequestStatus`, then sync the app's record to `Status` (and read
`Approver_ADUser` / `ApproveStep` for who just acted). This makes the handler
naturally correct across multiple callbacks: re-syncing to the same final
`Status` is a harmless no-op, and an intermediate `PENDING` keeps the request
open for the next level. Because it's a GET, the route must be a GET route, read
params from the query string, and needs **no** CSRF exclusion.

## Security: don't trust the GET blindly

This is the one thing not to gloss over. A callback that updates the database
purely on the incoming `status` query param is spoofable — anyone who learns the
URL and an `ExternalRefID` can approve their own request just by opening the
link, and because it's a plain GET, email/link scanners and browser prefetch can
even trigger it unintentionally. The existing production controller updates on
`status` with no verification. When you generate a callback handler, do **not**
copy that gap forward silently.

Instead, generate the handler with an explicit, prominent verification step and
a `// TODO(CAS): confirm how CAS signs/authenticates the callback with IT Dev`
marker. Offer the developer the options so they can pick what CAS actually
supports:

- **Signed query param / token** — CAS appends a signature or one-time token the
  app verifies against a secret in `.env` before acting. Preferred for a GET
  link.
- Drive the app's state from `RequestStatus.Status` rather than from the raw
  `status` param, and treat re-runs as idempotent — a legitimate second-level
  callback (or a prefetch/replay of one already applied) then just re-syncs to
  the same authoritative `Status` instead of double-applying or flipping a
  settled request. Don't hard-reject a repeat callback outright: for
  `ApproveType = 2` the second head's call is expected.

Make the verification the first thing the handler does, and make it loud in the
code that it's a placeholder until confirmed — never present an unverified
callback as finished.

## Getting the client credentials

Each app needs its own CAS `client_id` **and** `client_secret`. Tell the
developer to request them by contacting the **IT Dev team to register the app
for DKC OAuth**. Put both in `.env` (`CAS_CLIENT_ID=`, `CAS_CLIENT_SECRET=`),
never inline in code, and reference them from config so it's obvious what to fill
in. The secret is sent on `startApprove` to authenticate the app to CAS.

## How to generate

### 1. Pick the framework, then read its reference

Read the one file that matches the target stack — don't load all three:

- **Laravel** → `references/laravel.md`
- **Next.js** → `references/nextjs.md`
- **Any other backend** (Python/FastAPI/Django, Go, Node/Express, etc.) →
  `references/backend-generic.md` (the HTTP contract, framework-neutral)

If the stack is unclear from context, ask which one before generating.

### 2. Pick the generation mode

Ask (or infer from the request) which the developer wants:

- **Lean** — the service/client, a `startApprove` usage example, **and** a
  callback handler that does the CAS-specific work (verify the request, read the
  `status` / `ExternalRefID` query params, call `RequestStatus`, read
  `Approver_ADUser`), with the app's own record update left as clearly-marked
  TODOs. Good when the app already has its own approval bookkeeping.
- **Full scaffold** — everything in lean, plus the model/DB wiring, route
  registration, `.env` / config entries, and a result page/response. Good for a
  new integration from scratch.

The callback's CAS-specific logic (auth + parse + status fetch) is the reusable
part, so include it in **both** modes; the difference is only how much app-side
DB and routing you wire up. When in doubt, offer both briefly and let the
developer choose rather than guessing.

### 3. Generate, keeping these steadily in mind

- Use the app's own reference id for `ExternalRefID`; never reuse CAS's internal
  ids for it.
- Wrap both CAS calls in a timeout + try/catch and return a uniform
  `{ success, data, status }`-style result, so callers handle failures the same
  way. CAS is a network hop and will sometimes be slow or down.
- Keep secrets (`client_id`, `client_secret`, any callback secret) in env/config, never inline.
- Carry the security note from above into any callback handler you emit.
- Where a field's presence isn't confirmed, emit a `// TODO(CAS): ...` comment
  instead of assuming.

## Reference files

- `references/laravel.md` — `CASService`, callback controller, routes, `.env`,
  result view. Mirrors the shape already in production.
- `references/nextjs.md` — a CAS client module, a route handler that calls
  `startApprove`, and a callback route handler (App Router and Pages API notes).
- `references/backend-generic.md` — the raw HTTP contract and pseudocode so the
  pattern ports to any language/framework.
