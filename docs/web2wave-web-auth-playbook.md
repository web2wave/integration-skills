# web2wave Web Auth Integration Playbook

Framework-agnostic instructions for AI agents and developers integrating **web2wave** authorization into a **web application**.

Copy this file into your agent context (Cursor skill, Claude project doc, Codex instructions, etc.) and follow it end-to-end.

---

## 0. Before you start — ask the human

Do **not** invent secrets or skip these questions. Ask until answered:

1. **`WEB2WAVE_API_KEY`** — required. Never hardcode it. Server-only env only.
2. **Query parameter name** — default `web2wave_user_id`. Confirm if the project also accepts `user_id`.
3. **Entry URL(s)** — default `/signin?web2wave_user_id={guid}`.
4. **Email conflict policy** when a local user with the same email already exists:
   - `autologin` — link `web2wave_user_id` and sign them in
   - `merge` — same as autologin, plus copy attribution / properties into the existing row
   - `error` — reject with a conflict message  
   Default recommendation: `merge`.
5. **Post-login redirect** — product home / dashboard (project-specific).
6. **Which attribution fields to store** from `user_visits` — none required; pick UTMs / click IDs / geo as needed.
7. **Which quiz `properties` to map** into the local profile beyond `email` / `password` (optional).

```text
API base:  https://api.web2wave.com/api
Auth:      header api_key: <WEB2WAVE_API_KEY>
Docs:      https://docs.web2wave.com/llms.txt
```

---

## 1. Goal

When a user lands on:

```text
https://your-app.example/signin?web2wave_user_id={guid}
```

the backend must:

1. Resolve the web2wave user by GUID
2. Create or find a local user (email ± password from web2wave properties)
3. Persist `web2wave_user_id` on the local user
4. Write local id back as web2wave property `external_user_id`
5. Set `app_installed=1` **once**
6. Send event `"App installed"` **on every** link visit
7. Create a local session and redirect into the product
8. Show **all** subscriptions in Settings (with Manage link when present)

---

## 2. Security rules (non-negotiable)

- Call web2wave **only from the server**. Never ship the API key to the browser (`NEXT_PUBLIC_*` / `VITE_*` / etc.).
- Treat `password` from properties as a **bootstrap credential** — hash before storing; prefer “set a new password” later.
- Validate GUID/UUID shape before calling the API.
- Redirect after login so the GUID is not left in the address bar.
- Log failures without dumping the API key or raw passwords.
- Soft-fail web2wave side effects (property/event) after local user+session are created, unless the product requires hard-fail.

---

## 3. Data model (minimum)

| Field | Notes |
|-------|--------|
| `id` | Local primary key |
| `email` | From `user_email` / property `email` |
| `password_hash` | Nullable if no `password` property |
| `web2wave_user_id` | GUID from the URL — keep forever |
| `app_installed_reported` | Bool — whether `app_installed=1` was already set |
| `login_count` | Increment on each successful magic-link visit |
| `last_login_at` / `last_login_ip` / `last_user_agent` | Device / IP context for the `"App installed"` event |
| `attribution` (JSON, optional) | Snapshot from a `user_visits` row |

---

## 4. Live API contract (verified)

### 4.1 Auth & errors

- Header: `api_key: <key>` (not Bearer).
- Unknown user on properties often returns **HTTP 422**:

```json
{ "error": 1, "error_msg": "user not found" }
```

- Some error bodies use `error_msg`; treat `error === 1` as failure even if HTTP is 200.
- Treat **404 and 422** (and message matching `/not found/i`) as “web2wave user not found”.

### 4.2 `GET /user/properties?user={guid}`

Top-level keys:

```json
{
  "user_id": "00000000-0000-4000-8000-000000000001",
  "user_email": "user@example.com",
  "properties": [
    { "property": "email", "value": "user@example.com" },
    { "property": "password", "value": "optional-bootstrap" },
    { "property": "last_quiz_id", "value": "10001" },
    { "property": "last_quiz_path", "value": "/manage-subscription" },
    { "property": "app_installed", "value": "1" },
    { "property": "external_user_id", "value": "42" }
  ]
}
```

Rules:

- Email = `user_email` OR property `email` (there is usually **no** top-level `email`).
- Password = property `password` (often missing → create local user without password).
- Quiz id for events = property `last_quiz_id` (fallbacks: `quiz_id`, `last_quiz_path`, or `"app"`).

### 4.3 `GET /user/user_visits?user={guid}&limit=10`

Top-level keys include `user_id`, `user_email`, **`user_visits`**, `pagination`.

Useful visit fields (store what you need):

| Field | Example / notes |
|-------|------------------|
| `id` | Visit GUID |
| `created_at` | `2026-01-15 12:00:00` |
| `ip` | Client IP at quiz time |
| `url` | Landing URL |
| `user_agent` | Full UA |
| `user_platform` | e.g. `Mac` |
| `user_language` | e.g. `en` |
| `user_country_code` / `user_city_name` / `user_state_code` / `user_zip_code` | Geo |
| `utm_source` / `utm_medium` / `utm_campaign` / `utm_content` / `utm_term` | UTMs |
| `utm_id` / `utm_adset` / `utm_adname` / `utm_ad_id` / `utm_adset_id` | Ad metadata |
| `fbclid` / `gclid` / `ttclid` / `gbraid` | Click IDs |
| `visit_number` | Integer |
| `is_first_visit` | `1` / `0` |

### 4.4 `POST /user/properties?user={guid}`

```http
POST /user/properties?user={guid}
api_key: <key>
Content-Type: application/json

{ "property": "app_installed", "value": "1" }
```

Success example: `{ "result": "1" }`.

Also write:

```json
{ "property": "external_user_id", "value": "<local_user_id>" }
```

### 4.5 `POST /user/events?user={guid}`  ⚠️ strict

Live API **rejects** incomplete bodies:

| Body | Result |
|------|--------|
| `{ "event_name": "App installed" }` | 422 `The event value field is required.` |
| `+ event_value` | 422 `The quiz field is required.` |
| `+ quiz` | **200** |

Working payload:

```json
{
  "event_name": "App installed",
  "event_value": "1",
  "quiz": "10001",
  "user_agent": "Mozilla/5.0 …",
  "additional_data": [
    { "key": "ip", "value": "203.0.113.10" },
    { "key": "user_agent", "value": "Mozilla/5.0 …" },
    { "key": "login_count", "value": "1" },
    { "key": "platform", "value": "macOS" },
    { "key": "external_user_id", "value": "42" }
  ]
}
```

Rules:

- **`event_name`, `event_value`, and `quiz` are required.**
- `quiz` ← property `last_quiz_id` (or fallback `"app"`).
- Send this event on **every** successful magic-link visit (not only the first).
- `additional_data` is an array of `{key, value}` objects (not a free-form object).

### 4.6 Subscriptions

**Preferred:** `GET /subscriptions/list?user_id={guid}`  
Paginated Laravel-style response; rows are in **`data`**.

**Alternate:** `GET /user/subscriptions?user={guid}`  
Rows are in **`subscription`** (singular key).

Example row fields (verified):

```json
{
  "id": 100001,
  "user_id": "00000000-0000-4000-8000-000000000001",
  "user_email": "user@example.com",
  "status": "trialing",
  "paywall_name": "Acme Premium",
  "plan_name": "acme_premium",
  "price_label": "9.99 USD / 1 month → 19.99 USD / 1 month",
  "next_charge_date": "2026-02-15 12:00:00",
  "canceled_at": null,
  "manage_link": "",
  "currency": "usd",
  "amount_real": "9.99",
  "payment_system_label": "stripe"
}
```

Mapping for Settings UI:

| UI | Field | Notes |
|----|-------|-------|
| Status | `status` | Show **all** statuses |
| Product | `paywall_name` → `plan_name` → `price_label` → `quiz_name` | |
| Until | `next_charge_date` | If canceled, prefer `canceled_at` |
| Manage | `manage_link` | May be `""` for some payment providers — **hide** button when blank |

Entitlement (access control, not UI filter):

- Paid / skip paywall if any sub has `status ∈ {active, trialing}` (also accept spelling `trialling` if seen).
- Still **list** canceled / past_due / paused / etc. in Settings.

---

## 5. Server flow (pseudocode)

```text
ON GET/POST /signin?{QUERY_PARAM}={guid}

  IF missing WEB2WAVE_API_KEY → 500 config error
  IF guid not UUID-like → 400

  propsRes = GET /user/properties?user=guid
  IF fail with not-found (404/422) → 404 "web2wave user not found"
  IF other fail → 502

  email = props.user_email OR property(props, "email")
  IF no email → 422

  passwordPlain = property(props, "password")          // may be null
  quizId = property(props, "last_quiz_id")
           OR property(props, "quiz_id")
           OR property(props, "last_quiz_path")
           OR "app"

  visitsRes = GET /user/user_visits?user=guid&limit=10   // best-effort
  attribution = pickVisitFields(visitsRes.user_visits)  // optional

  local = findByWeb2waveUserId(guid) OR findByEmail(email)

  IF local is null:
      local = createUser(email, hash?(passwordPlain), guid, attribution)
  ELSE IF local.web2wave_user_id is null:
      APPLY email_conflict_policy (autologin|merge|error)
  ELSE IF local.web2wave_user_id != guid:
      → 409 identity mismatch
  // else returning user

  local.login_count += 1
  local.last_login_* = request meta
  save(local)

  // Soft-fail block — still redirect even if these fail
  POST property external_user_id = string(local.id)

  IF NOT local.app_installed_reported:
      POST property app_installed = "1"
      local.app_installed_reported = true
      save(local)

  POST /user/events?user=guid
    {
      event_name: "App installed",
      event_value: "1",
      quiz: quizId,
      user_agent: request.userAgent,
      additional_data: [
        { key: "ip", value: request.ip },
        { key: "user_agent", value: request.userAgent },
        { key: "login_count", value: string(local.login_count) },
        { key: "platform", value: detectPlatform(request.userAgent) },
        { key: "external_user_id", value: string(local.id) }
      ]
    }

  createSession(local)
  redirect(POST_LOGIN_PATH)   // strips query string
```

Idempotency:

- `app_installed=1` → once per local user (`app_installed_reported`).
- `"App installed"` event → **every** visit.
- Retry `app_installed` on next visit if the flag is still false after a soft-fail.

---

## 6. Settings / profile UI

Show **every** subscription from `/subscriptions/list` (or `/user/subscriptions`).

Suggested row:

```text
{status} · entitled?: {product_name}
Until: {until_date}     [Manage]
```

- Mark entitled when `status` is `active` or `trialing`.
- Hide `[Manage]` when `manage_link` is null/empty.
- Optionally show a read-only attribution JSON snapshot from the stored visit.

Refresh from API on each Settings view (cache optional).

---

## 7. Configuration checklist

```bash
WEB2WAVE_API_KEY=                 # required, server-only
WEB2WAVE_API_BASE=https://api.web2wave.com/api
WEB2WAVE_QUERY_PARAM=web2wave_user_id
WEB2WAVE_EMAIL_CONFLICT_POLICY=merge   # autologin | merge | error
WEB2WAVE_POST_LOGIN_REDIRECT=/dashboard
AUTH_SECRET=                      # session signing
COOKIE_SECURE=false               # true only behind HTTPS
```

Funnel “after pay” / install link in the web2wave project:

```text
https://YOUR_DOMAIN/signin?web2wave_user_id={user_id}
```

---

## 8. Acceptance tests

1. Fresh GUID with email + password → local user created, password hashed, session set, redirect works.
2. Fresh GUID with email only → user created with `password_hash = null`, still signed in.
3. Same GUID again → no duplicate; `login_count` increments; event sent again; `app_installed` **not** re-POSTed if already reported.
4. Existing local email + `merge`/`autologin` → linked; `error` → 409.
5. Invalid GUID → 400; unknown web2wave user (422/404) → 404 page.
6. Event without `event_value` or `quiz` must not be sent (API will 422).
7. Settings lists all statuses (including `trialing`); Manage hidden when `manage_link` is empty.
8. After first login, properties include `app_installed=1` and `external_user_id=<local id>`.
9. API key never appears in client bundles / HTML / public env.

---

## 9. Optional follow-ups (recommend, don’t block)

- Webhooks for `subscription.*` so access stays fresh without waiting for next login: https://docs.web2wave.com/reference/webhook-formats
- Skip in-app onboarding when quiz answers already exist in `properties`
- Map selected quiz answers into the local profile
- First-touch vs last-touch attribution from `user_visits`
- Force password reset after bootstrap-password login

---

## 10. Reference implementation

This repository ships a minimal Next.js example under [`ref-app/`](../ref-app/) (SQLite, cookie session, `/signin`, `/settings`) that follows this playbook.

Use it as a working example, not as a mandatory stack.

---

## 11. Agent output expectations

When applying this playbook to another codebase, the agent should:

1. Ask for the API key and the decisions in §0
2. Add server-only env + `.env.example` (no real secrets)
3. Implement §5 using the **live field names** in §4 (not OpenAPI-only names)
4. Add Settings UI per §6
5. Document the entry URL to paste into web2wave project settings
6. Run / describe the acceptance checks in §8
