---
name: dkc-oauth-message
description: >-
  Send notifications through DKC's central message gateway at
  https://oauth.dhammakaya.network/api/send-message — a single POST that fans one
  message out over Email, LINE text, or LINE Flex, addressing people either
  directly (`to`) or by AD username (`toAd`, resolved against the directory). Use
  this whenever an app (Laravel, Next.js, any backend) needs to email or LINE
  someone in the organization — alerts, reminders, status updates — or whenever
  the user mentions send-message, app_name / channel / to / toAd / email_body /
  line_text / line_flex, a returned tracking_id, or building LINE Flex Message
  JSON for an internal system. Reach for it on Thai phrasings too:
  "ส่งแจ้งเตือนเข้าไลน์", "ยิงอีเมลจากระบบ", "ต่อ API ส่งข้อความ", "ทำ noti ให้ผู้ใช้".
  Siblings on the same host: dkc-oauth-login (signing users in) and
  dkc-cas-approval (supervisor approval — CAS notifies the supervisor itself, so
  never rebuild an approval flow out of this API).
---

# DKC Message Gateway (send-message)

DKC runs one notification endpoint for internal apps. An app POSTs a message
once; the gateway delivers it over Email, LINE text, LINE Flex, or email + LINE
together, and resolves recipients from AD usernames so apps never store anyone's
address or LINE id.

This skill exists because the payload rules aren't guessable: required fields
depend on `channel`, `line_text` and `line_flex` are mutually exclusive,
recipients are addressed two different ways with different consequences, and a
`200` means less than it looks like.

## The call

`POST https://oauth.dhammakaya.network/api/send-message`

```
Content-Type: application/json
Accept: application/json
```

**Authentication is `client_id` + `client_secret` in the JSON body** — the same
credential model as CAS. Despite what the endpoint's OpenAPI annotation says
about bearer auth, there is no token to fetch and no `Authorization` header: the
credentials ride in the payload of every request. Don't generate a token-exchange
step for this API.

That has one consequence to handle deliberately: **the secret is in the request
body, so never log the payload as-is.** A generated service that dumps the full
request on error will write `client_secret` into the app's log files, and from
there into whatever ships logs elsewhere. Log the fields that help debugging
(`app_name`, `channel`, recipients, response) and redact or omit the credentials.

### .env

```env
MSG_ENDPOINT="https://oauth.dhammakaya.network/api/send-message"
MSG_APP_NAME=
MSG_CLIENT_ID=
MSG_CLIENT_SECRET=
```

`MSG_APP_NAME` is the app's own identifier that goes in every payload as
`app_name` — it's how the gateway attributes a message, so pick a stable value
(`NEXTCLINIC`, `WAREHOUSE`) and don't change it casually. If the app already
integrates DKC OAuth login, ask whether the same client credentials are valid
here before adding a second pair.

## Request body

| Field | Req? | Meaning |
|---|---|---|
| `client_id` | required | The app's registered client id. From config, never inline. |
| `client_secret` | required | The matching secret. Same. |
| `app_name` | required | Source application name, e.g. `MY_APP`. |
| `subject` | required | Message subject. Used as the **email** subject line. |
| `message` | required | Always include it. If the app has nothing specific to put here, fall back to the default below. |
| `channel` | optional | Which channels to deliver on. Omitting it means **all** of them — see below, and always set it explicitly. |
| `to` | conditional | Array of literal destinations (email addresses). Required when `toAd` is absent. |
| `toAd` | conditional | Array of **AD usernames**; the gateway resolves address / LINE id itself. Required for the LINE channel. |
| `email_body` | conditional | Message body, shared by the `email` **and** `hr_mobile` channels. **HTML is accepted.** Required whenever the resolved set includes either. |
| `line_text` | conditional | Plain LINE message. **Never send together with `line_flex`.** |
| `line_flex` | conditional | A complete Flex Message object. **Never send together with `line_text`.** |
| `email_source` | optional | `hr` or `ad`. In practice leave it out — the default is fine. |

### Always send `message`

Every payload carries a `message` field, including when the app has nothing
meaningful to put in it. When there's no app-supplied value, fall back to:

```
DKC OAuth Msg <app_name>
```

Build the fallback in the client rather than leaving the caller to remember it —
a required field that callers fill in by hand is a field that eventually arrives
empty from the one code path nobody tested. Putting `app_name` in the fallback
also means a message traced back later says which app sent it, instead of every
app's messages looking alike.

`TODO(MSG)`: what the gateway does with `message` isn't documented — whether it's
a log label, an internal title, or shown to the recipient. Until that's
confirmed, don't put anything in it that would embarrass the app if a recipient
saw it, and don't rely on it being displayed either.

## Channels

`channel` accepts a string **or an array**, and may be omitted entirely:

| Value | Delivers on |
|---|---|
| omitted / `null` | **everything** — email + LINE + `hr_mobile` (the org's internal mobile app) |
| `"all"` | same as omitting it |
| `"both"` | email + LINE only (legacy shorthand, predates `hr_mobile`) |
| `"email"` / `"line"` / `"hr_mobile"` | that one channel |
| `["email","hr_mobile"]` | exactly those |

**Always set `channel` explicitly.** Leaving it out is the one mistake in this
API that costs real money and real goodwill: the field is optional, so a
developer who forgets it gets no error and no warning — the gateway simply
delivers on every channel to every recipient. A test run against a list of 200
staff quietly reaches all of them three times over — inbox, LINE, and an in-app
push — which is the kind of mistake people remember. Generated code should always
send the field, and a message helper should require it as a parameter rather than
defaulting it.

Prefer the array form for new code. `"both"` still works, but it means
"email + LINE" — it predates `hr_mobile` and does *not* mean "everything", which
is exactly the wrong guess for someone reading it later. `["email","line"]` says
the same thing without the ambiguity.

Required fields follow from the resolved channel set:

- set includes `email` **or** `hr_mobile` → `email_body` required (both channels
  read the same field)
- set includes `line` → exactly one of `line_text` / `line_flex`, and `toAd`
  required
- set includes `hr_mobile` → `toAd` required

Only `email` can be addressed with `to`. Both `line` and `hr_mobile` resolve the
recipient through the directory, so a payload that names people only by `to` and
asks for either channel is a validation failure — see below.

### `hr_mobile` shares `email_body`

`hr_mobile` delivers to the organization's own internal mobile app — not SMS,
so there's no per-message cost and no 160-character worry. It has no field of
its own: it sends whatever is in `email_body`, and it reaches the app account
linked to the AD username in `toAd`, so it needs `toAd` just as LINE does.

The thing to watch is markup. `email_body` accepts HTML, and a body written for
an email client is unlikely to render the same way in the app — at best the
formatting is ignored, at worst the tags show up as literal text. `TODO(MSG)`:
whether the gateway strips HTML for this channel is unconfirmed.

Two workable patterns, in order of preference:

- **Keep the shared body plain** when the same wording suits both, and send one
  call with `channel: ["email","hr_mobile"]`. Simplest, one `tracking_id`, one
  failure mode.
- **Split into two calls** when the email genuinely needs HTML — `["email"]`
  with the formatted body, `["hr_mobile"]` with a short plain one. The cost is a
  second call and a second thing that can fail, so don't do it reflexively.

So validation can't switch on a single string — resolve `channel` into a set
first, then check the rules against that set. Code that pattern-matches on
`'email' | 'line' | 'both'` breaks the moment someone passes an array.

### Addressing: prefer `toAd`

`toAd` is the right default in almost every case, for a reason worth telling
developers rather than just asserting: it keys notifications on the same AD
username that `dkc-oauth-login` returns as `username` and CAS calls
`Requester_ADUser`. One identity runs through login, approval, and notification,
so someone who changes email or re-links LINE keeps receiving messages with no
change on the app's side. Storing raw addresses in the app duplicates the
directory and quietly rots.

`to` reaches the email channel and nothing else. Neither LINE nor the mobile app
can be addressed by a literal string, because both resolve their destination
through the directory entry behind an AD username. That's worth keeping in mind
when someone asks to "just send it to this address as well" on a multi-channel
notification: that person will receive the email only.

Use `to` only when the recipient genuinely isn't an org account — an outside
vendor, a shared mailbox, a test address.

**An unresolvable `toAd` is skipped silently.** The gateway drops that one
recipient and delivers to the rest; the response still says `success` and says
nothing about who was skipped. So a typo'd username, a resigned staff member, or
someone who never linked LINE produces a message that simply never arrives, with
no error anywhere. Two things follow:

- Validate usernames against the app's own user table (or the profile it got
  from OAuth login) *before* sending, rather than trusting free-typed input.
- For anything that must reach the person — deadlines, approvals, credentials —
  include `email` in the channel set, so a missing LINE link still leaves email
  as the path.

### `email_body` is HTML

Render it from a template (Blade view, React email, whatever the stack has)
rather than concatenating strings, and **escape any user-supplied text going into
it**. A notification that interpolates a free-text field — a rejection reason, an
item name — is an HTML injection straight into people's inboxes, and unlike a web
page there's no CSP or framework auto-escaping standing in the way.

### `line_flex` must be a complete Flex message

Wrap the bubble in the full envelope; the gateway does not add it:

```json
{
  "type": "flex",
  "altText": "ใบขอซื้อ WO022665 ได้รับอนุมัติ",
  "contents": { "type": "bubble", "body": { "...": "..." } }
}
```

`altText` isn't decorative — it's what shows in the LINE notification banner and
chat list preview, and what a user on an unsupported client sees instead of the
card. Generate a real summary sentence there, never `"Flex Message"` or the
app's name.

Flex JSON is validated by LINE, not by the app, so a malformed bubble is rejected
at delivery time where the app may never see the reason. Build it from a small
helper that produces the structure — not string concatenation — and keep one
known-good template to modify. Use `line_text` when the message is one sentence;
Flex is worth it only for structured content (a record with fields, an action
button).

## Response

Success (`200`):

```json
{ "status": "success", "tracking_id": "MSG123456" }
```

**Log `tracking_id` next to the app's own record id.** It's the only handle for
answering "did the notification actually go out?" later, and a support question
about a missing message is unanswerable without it.

A 200 means the gateway accepted the message — not that anyone received it. It
covers neither the downstream SMTP/LINE push result nor recipients that were
skipped for being unresolvable. Render "ส่งคำขอแจ้งเตือนแล้ว", not
"แจ้งเตือนเรียบร้อย".

### Errors

| Status | Body | What it means |
|---|---|---|
| `422` | `{"errors": {"line_text": ["Either line_text or line_flex must be provided for LINE channel."]}}` | Laravel-style validation map keyed by field. **Never retry** — the payload is wrong and will stay wrong. Log the whole `errors` object; the key names the field. |
| `401` | `{"error": "Unauthorized"}` | Wrong or missing `client_id` / `client_secret`. This is a configuration error, not a transient one — surface it loudly and stop. Retrying can't fix it. |

Anything else (5xx, timeout, connection reset, an HTML error page) is *unknown
outcome*, not failure: the message may have gone out. Keep that as a third
distinct case — collapsing it into "failed" and retrying is how one approval
notification becomes four LINE messages to a supervisor at 2am.

## Operational guidance

**Send outside the request cycle.** Put the call in a queued job / background
task. The gateway is a network hop; a user's save shouldn't fail or hang because
a LINE push was slow. If the app has no queue, at minimum give the call a short
explicit timeout and swallow failures into a logged error rather than a 500.

**Retries can duplicate.** There's no idempotency key, so a retry after a timeout
may deliver twice. Retry only on connection errors and 5xx — never on 422 or 401.
Record `tracking_id` (or a "sent" flag on the app's record) before any retry
decision, so a re-run of the job doesn't resend.

**One call for many recipients.** `to` and `toAd` are arrays with no documented
limit; send twenty people in one request rather than looping twenty requests.
Looping multiplies the failure modes and the duplicate risk for a single logical
notification.

**Keep the credentials server-side.** No `NEXT_PUBLIC_*`, no client component, no
fetch from the browser. Since the secret travels in the body, a front-end call
hands the org's notification gateway to anyone who opens devtools — a spam relay
sending under the organization's name.

**Don't put secrets or sensitive personal data in message bodies.** They persist
in mailboxes, chat history, and gateway logs, all beyond the app's reach to
redact later.

## Still unconfirmed

Emit `// TODO(MSG): ...` rather than guessing on these:

- **What `message` is used for** — log label, internal title, or shown to the
  recipient.
- **Whether `tracking_id` can be queried** for delivery status anywhere.
- **Whether HTML in `email_body` is stripped for `hr_mobile`,** or rendered, or
  shown as literal tags in the app.
- **Whether `to` accepts anything beyond email addresses.** Treat it as
  email-only.
- **Exact credential field names in the body** if the gateway names them
  something other than `client_id` / `client_secret` — check against a working
  call before shipping.

## Getting credentials

`MSG_CLIENT_ID` / `MSG_CLIENT_SECRET` come from registering the app with the **IT
Dev team**, the same route as CAS and OAuth login. Ask at the same time for a
test recipient, because there's no dry-run mode: the first end-to-end test sends
a real email and a real LINE message to a real person.

## How to generate

### 1. Pick the framework, then read its reference

Read only the file matching the target stack:

- **Laravel** → `references/laravel.md`
- **Next.js** → `references/nextjs.md`
- **Anything else** (FastAPI/Django, Express, Go, ASP.NET, n8n…) →
  `references/backend-generic.md`

If the stack isn't clear from context, ask before generating.

### 2. Pick the depth

- **Lean** — a message client/service with one `send()` that handles credentials,
  payload assembly, the channel rules, and error mapping. Right when the app just
  needs to fire a notification from code it already has.
- **Full scaffold** — the above plus config/`.env` entries, a queued job wrapper,
  a Flex template helper, and somewhere to store `tracking_id`. Right for a first
  integration or when notifications will be a recurring feature.

The payload-and-error half is the reusable part and belongs in both.

### 3. While generating, keep these in mind

- Resolve `channel` into a set first, then validate against it **before** the
  HTTP call — `email_body` when the set has `email` or `hr_mobile`, exactly one
  of `line_text`/`line_flex` plus `toAd` when it has `line`, at least one of
  `to`/`toAd` always. Catching it locally turns a 422 round-trip into a clear
  exception at the call site.
- Return a uniform result (`{ success, tracking_id, status, errors }`) so callers
  handle every outcome the same way.
- Read credentials and `app_name` from config; redact the credentials from logs.
- Default `message` inside the client so it's never absent, rather than asking
  callers to pass it every time.
- Don't build a per-channel API surface (`sendEmail()`, `sendLine()`) that fires
  a call each — one request with `channel: ["email","line"]` does it in one, and
  separate calls mean separate failure modes for one notification.
- If the app also signs users in or needs supervisor approval, mention that
  `dkc-oauth-login` and `dkc-cas-approval` cover those, and that the AD
  `username` is the field all three share.

## Reference files

- `references/laravel.md` — config, `DkcMessageService`, queued job, Flex helper,
  usage.
- `references/nextjs.md` — a server-only client module, route handler usage, and
  why this must never run in a client component.
- `references/backend-generic.md` — the raw HTTP contract, payload matrix, and
  framework-neutral pseudocode.
