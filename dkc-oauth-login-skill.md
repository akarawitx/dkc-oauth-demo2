---
name: dkc-oauth-login
description: >-
  Wire an app's user login / SSO to DKC's OAuth 2.0 server at
  oauth.dhammakaya.network — the authorization-code flow (redirect to
  /oauth/authorize, exchange the code at /oauth/token, read the profile from
  /api/user) that lets staff sign in with their organization account. Use this
  whenever the user is adding, fixing, or reviewing sign-in for any app
  (Laravel, Next.js, any backend) that authenticates against the org account, or
  mentions OAUTH_BASE_URI, OAUTH_CLIENT_ID/SECRET, OAUTH_REDIRECT_URI, a
  callback route reading ?code=, trading an authorization code for an access
  token, or reading username / display_name / line_internal_id from /api/user —
  even if they never say "DKC OAuth". Applies to Thai phrasings too: "ทำ login
  ด้วย OAuth องค์กร", "ต่อ SSO", "เขียน route callback รับ code". This skill is
  about who the user is (login); routing an approval to a supervisor
  (startApprove, ApproveType, HeadKong) is the sibling skill dkc-cas-approval —
  same host, different service.
---

# DKC OAuth Login (SSO)

DKC runs an OAuth 2.0 authorization server at `https://oauth.dhammakaya.network`
so internal apps don't each keep their own passwords. It behaves like a standard
Laravel Passport server: the app redirects the browser there, the user signs in
with their organization account, and the app gets back an authorization code it
exchanges server-side for an access token, then reads the profile. This skill is
only about signing users in.

## The flow (what you're implementing)

1. **Redirect** — the app sends the browser to `/oauth/authorize` with its
   `client_id`, the exact registered `redirect_uri`, `response_type=code`, and a
   `state` value.
2. **User signs in** on the OAuth server and approves.
3. **Callback** — the server redirects back to the app's `redirect_uri` with
   `?code=...&state=...`. The app verifies `state` matches what it stored, then
   discards the stored value.
4. **Exchange** — the app POSTs the code to `/oauth/token` **from the server**
   (never the browser — the client secret is involved) and gets an
   `access_token`.
5. **Profile** — the app calls `GET /api/user` with `Authorization: Bearer
   <access_token>`, maps the returned identity to its own user record, and
   starts its own session.

After step 5 the OAuth server's job is done. The app's own session governs the
rest of the visit; the access token is only needed again if the app calls other
DKC APIs on the user's behalf.

## Shared contract (framework-independent)

### .env

```env
OAUTH_BASE_URI="https://oauth.dhammakaya.network"
OAUTH_CLIENT_ID=
OAUTH_CLIENT_SECRET=
OAUTH_REDIRECT_URI=
```

Leave the last three blank in committed files — they're per-app values issued at
registration (see "Getting credentials"). Read them through the framework's
config layer rather than calling `env()` deep inside code, so cached-config
environments don't silently get empty strings.

### Endpoints

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/oauth/authorize` | Browser redirect. Query: `client_id`, `redirect_uri`, `response_type=code`, `state`. **`scope` is not required — omit it.** |
| `POST` | `/oauth/token` | Server-to-server. Body: `grant_type=authorization_code`, `client_id`, `client_secret`, `redirect_uri`, `code`. Returns JSON with `access_token`, `refresh_token`, and typically `token_type` / `expires_in`. |
| `POST` | `/oauth/token` | Also the refresh endpoint: `grant_type=refresh_token`, `client_id`, `client_secret`, `refresh_token`. Returns a fresh token pair. |
| `GET` | `/api/user` | Send `Authorization: Bearer <access_token>` and `Accept: application/json`. Returns the signed-in user's profile. |
| `GET` | `/logout` | Ends the user's session **at the OAuth server**. Takes `?redirect_url=` (URL-encoded) to send the user back afterwards. This is a **browser redirect, not an API call** — send the user there; don't fetch it from the backend. |

`redirect_uri` must be **byte-identical** in the authorize request and the token
request, and identical to the value registered with IT Dev — scheme, host, port,
trailing slash and all. A mismatch is the single most common cause of
`invalid_client` / `invalid_grant` at the token step, so when a developer reports
that error, check this before anything else.

### /api/user response

```json
{
  "id": 12,
  "name": "somchai",
  "email": "somchai@example.org",
  "email_verified_at": null,
  "created_at": "2024-07-22T07:56:18.870000Z",
  "updated_at": "2024-07-22T07:56:18.870000Z",
  "username": "somchai",
  "display_name": "สมชาย ใจดี",
  "offices": [
    {
      "position": "หัวหน้ากอง/ศูนย์",
      "kong": "กองบริการสารสนเทศ",
      "section": "ฝ่ายสารสนเทศ",
      "samnak": "สำนักปธ.คกก.บริหาร"
    }
  ],
  "line_internal_id": "U1234...",
  "line_name": "somchai",
  "line_picture": "https://..."
}
```

| Field | How to use it |
|---|---|
| `id` | The OAuth server's own user id. Stable, but scoped to that server. |
| `username` | **The identity key.** This is the AD username, and the same value CAS uses for `Requester_ADUser` / `Approver_ADUser`, so keying local records on it makes the app line up with the rest of the org's systems. |
| `display_name` | Full Thai display name — use this for greetings, headers, audit trails. |
| `name` | Short/login-style name. Not a good display label and not guaranteed unique-looking; prefer `display_name` for UI and `username` for identity. |
| `email` | Contact address. Do **not** join on it — email can change or be blank, and matching on it lets a reassigned address inherit an account. |
| `offices` | **Array** of the user's org positions — a person can hold more than one. Each entry has `position` (title, e.g. หัวหน้ากอง/ศูนย์), `kong` (กอง/ศูนย์), `section` (ฝ่าย), and `samnak` (สำนัก). Use it to show department and to tell whether the user *is* a supervisor. Same org hierarchy CAS routes approvals through — see the note below. |
| `line_internal_id` | The user's LINE id as known to the org. Valuable: an app that wants to push LINE notifications already has it here, with no separate LINE binding step. |
| `line_name`, `line_picture` | LINE display name and avatar URL. Handy for profile UI, but treat as optional — a user who never linked LINE may have these null. |
| `email_verified_at`, timestamps | OAuth-server bookkeeping. Not meaningful to the app. |

Treat every field except `id` and `username` as possibly null and code
defensively — the shape above is what a fully-populated account looks like, not a
guarantee. `offices` in particular may be an empty array for a user who holds no
listed position, so never assume `offices[0]` exists.

### offices and CAS

The `kong` / `section` / `samnak` / `position` fields are the same org structure
the CAS approval service (`dkc-cas-approval`) uses to decide who approves a
request — `HeadKong` is the head of a `kong`, `HeadSamnak` the head of a
`samnak`. Two consequences worth acting on:

- An app can read `offices` at login to learn a user's department for display or
  filtering, and to tell whether they hold a `หัวหน้า` / `รักษาการหัวหน้า`
  (head / acting-head) position — useful for showing supervisor-only UI without a
  second lookup.
- Because both services key on `username` and share this hierarchy, an app doing
  login **and** approvals can capture `offices` here and hand the right `kong` /
  `samnak` to CAS, instead of asking the user to pick their department by hand.

Store `offices` as its own related table or a JSON column rather than flattening
it onto the user row — a single row can't represent someone who heads two units,
as the example above (substantive head of one กอง plus acting head of another)
shows.

### UTF-8 (Thai text)

`display_name` and every `offices` field are Thai. The JSON response is already
UTF-8 and any standard parser decodes it correctly — so when Thai turns into
`????`, `à¸ª...`, or `\u0e2b...`, the bug is never the OAuth call itself, it's a
layer downstream that wasn't told the data is UTF-8. Check these:

- **Database must be `utf8mb4`** — charset *and* collation on the column, *and*
  the DB connection. MySQL's legacy `utf8` and a `latin1` connection are the
  usual culprits; they store Thai as `????`, and it's unrecoverable after the
  write, not just a display glitch. `utf8mb4` end to end is the fix.
- **Cookies and tokens can't hold raw non-ASCII.** If `display_name` or
  `offices` goes into a cookie or a signed token, encode it first (a JWT's
  base64 or `encodeURIComponent` — a plain `Set-Cookie` with raw Thai is
  dropped or corrupted by the browser).
- **Re-serializing for logs or APIs** — a serializer that escapes non-ASCII
  gives `\u0e2b...`; harmless but unreadable. Turn on the "don't escape unicode"
  flag if you want legible Thai in logs (`JSON_UNESCAPED_UNICODE` in PHP, the
  default in JS `JSON.stringify`).
- **Your own responses** need `Content-Type: ...; charset=utf-8`, or a page that
  echoes the name renders mojibake even when storage was fine.

The per-framework references show the concrete settings.

## Security: the parts worth not getting wrong

**The server treats `state` as optional — send it anyway, every time.** Without
it, an attacker can feed a victim's browser a callback URL carrying the attacker's authorization code and silently log the
victim into the attacker's account (session fixation via login CSRF). Generate a
cryptographically random `state`, store it in the session before redirecting,
compare on callback, and delete it after one use. If it's missing or doesn't
match, abort — don't "fall back" to accepting the code.

**The token exchange belongs on the server.** `OAUTH_CLIENT_SECRET` must never
reach the browser: not in JavaScript, not in a `NEXT_PUBLIC_*` variable, not in
a client component. Any generated code that would ship it to the client is a
defect, not a style preference.

**Don't trust the callback's query string beyond the code.** The only things to
read from it are `code`, `state`, and possibly `error`. Everything about *who
the user is* comes from `/api/user` over the token — never from a parameter the
browser could have edited.

**Decide the "unknown user" policy explicitly.** When a valid org user signs in
but has no record in the app, the app either (a) auto-creates one (JIT
provisioning) or (b) refuses with "your account isn't authorized for this app".
Both are legitimate and the difference is a real access-control decision, so ask
the developer which one they want rather than defaulting silently — and if they
haven't decided, generate the safer refusal path with a clear TODO.

**Regenerate the session id** after a successful login, so a pre-login session
identifier can't be reused afterwards.

## Refresh tokens

`/oauth/token` returns a `refresh_token` alongside the access token. Whether the
app needs it depends on what it does after login:

- **Login only** — the app reads the profile once, creates its own session, and
  never calls DKC again. Then don't store either token. Persisting a refresh
  token the app will never use is a liability with no upside.
- **Keeps calling DKC APIs on the user's behalf** — store the refresh token
  server-side and encrypted at rest (Laravel's `encrypted` cast, or equivalent),
  never in a cookie or anywhere the browser can see. When a call returns 401,
  refresh once and retry; if the refresh also fails, clear the session and send
  the user back through login rather than looping.

Refresh with `grant_type=refresh_token` plus `client_id`, `client_secret`, and
`refresh_token`. Expect a **new** refresh token in the response and replace the
stored one — assuming the old one stays valid is how apps end up logging
everyone out a week later.

## Logout

App logout has two halves, **in this order**:

1. Clear the app's own session (invalidate, regenerate the CSRF token).
2. Redirect the browser to `GET {BASE}/logout?redirect_url=<app login page>` so
   the OAuth server's session ends too and the user lands back on the app.

Skip step 2 and the user "logs out", clicks login, and is instantly back in
without typing a password — because the OAuth server still recognizes them. On a
shared or public machine that's a real exposure, not just a confusing UX.

**Step 2 is a redirect, not an HTTP call from the backend.** The session being
ended lives in the user's browser cookies for `oauth.dhammakaya.network`, so a
server-side `Http::get()` / `fetch()` sends the wrong cookies (or none) and ends
nothing while appearing to succeed. The same applies on the client: `fetch()`ing
the logout URL from JavaScript doesn't work either — the browser has to actually
navigate there.

So the local cleanup must finish *before* the redirect, since once the response
is a redirect the app's code is done running for that request.

### `redirect_url`

URL-encode the value, and **build it from the app's own config — never from
user-supplied input.** Passing a `?next=` query parameter straight through would
turn the org's logout endpoint into an open redirect that sends users to an
attacker's page from a trusted domain. A constant like `route('login')` or
`APP_URL + "/login"` is what belongs here.

If the parameter appears to be ignored, ask IT Dev whether the target has to be
pre-registered alongside the app's `redirect_uri` — that's the usual reason a
post-logout URL silently doesn't take.

## Still unconfirmed

- **PKCE** — the flow works as a confidential client with a secret. Whether the
  server also accepts PKCE only matters for public clients (mobile/SPA), so ask
  IT Dev before building one that way.

## Getting credentials

`OAUTH_CLIENT_ID`, `OAUTH_CLIENT_SECRET`, and the registered `OAUTH_REDIRECT_URI`
come from **registering the app for DKC OAuth with the IT Dev team**. The
redirect URI is fixed at registration, which has a practical consequence worth
flagging early: local development needs its own registered URI (e.g.
`http://vm13:8003/callback` or a localhost port), or developers can't test the
flow at all. Raise that when scaffolding a new integration rather than after the
first `invalid_client`.

## How to generate

### 1. Pick the framework, then read its reference

Read only the file matching the target stack:

- **Laravel** → `references/laravel.md`
- **Next.js** → `references/nextjs.md`
- **Anything else** (FastAPI/Django, Express, Go, ASP.NET…) →
  `references/backend-generic.md`

If the stack isn't clear from context, ask before generating.

### 2. Pick the depth

- **Lean** — the OAuth client/service plus the redirect and callback handlers
  (state, exchange, profile fetch), with local user lookup and session creation
  left as marked TODOs. Right when the app already has its own auth plumbing and
  is just swapping the identity source.
- **Full scaffold** — the above plus routes, config/`.env` entries, the local
  user mapping, session/middleware wiring, and a logout route. Right for a new
  app or a first integration.

The OAuth-specific half (state → exchange → profile) is the reusable part and
belongs in both; the difference is only how much app-side wiring comes with it.
When in doubt, offer both in a sentence and let the developer pick.

### 3. While generating, keep these in mind

- Give both HTTP calls an explicit timeout and error handling. The OAuth server
  is a network hop; a hung token exchange should surface as a clean "login
  failed, try again", not a white screen.
- On any failure — bad `state`, `error=access_denied`, non-200 token response,
  unusable profile — send the user to a login page with a plain message and log
  the detail server-side. Don't render raw OAuth error bodies to users.
- Key local records on `username` (or `id`), never on `email`.
- Keep credentials in config; never inline, never client-side.
- Where the contract is unconfirmed (above), emit `// TODO(OAUTH): ...` rather
  than guessing.
- If the app also needs supervisor approvals, mention that `dkc-cas-approval`
  covers it and that `username` here is the same value CAS calls
  `Requester_ADUser` — the two integrations fit together on that field.

## Reference files

- `references/laravel.md` — config, routes, `DkcOAuthService`, redirect +
  callback controller, user mapping, logout.
- `references/nextjs.md` — App Router route handlers for login/callback, cookie
  handling for `state` and session, and notes on doing it via Auth.js instead.
- `references/backend-generic.md` — the raw HTTP contract and framework-neutral
  pseudocode.
