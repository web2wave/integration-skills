# web2wave Web Auth Integration Playbook
Framework-agnostic instructions for AI agents and developers integrating **web2wave** authorization into a **web application**.
Copy this file into your agent context (Cursor skill, Claude project doc, Codex instructions, etc.) and follow it end-to-end.
---
## 0. Before you start — ask the human
Do **not** invent secrets or skip these questions. Ask until answered:
1. **`WEB2WAVE_API_KEY`** — required. Never hardcode it in source. Put it in server-only env (e.g. `WEB2WAVE_API_KEY`).
2. **Query parameter name** — default `web2wave_user_id`. Confirm if the project prefers `user_id` or both.
3. **Entry URL(s)** — e.g. `/signin`, `/auth/web2wave`. Default: `/signin?web2wave_user_id={guid}`.
4. **Email conflict policy** when a local user with the same email already exists:
   - `autologin` — link `web2wave_user_id` and sign them in
   - `merge` — same as autologin plus copy attribution / properties into the existing row
   - `error` — reject and show a conflict message  
   Default recommendation: `merge` (configurable).
5. **Post-login redirect** — where to send the user after a successful session (product home / dashboard). Project-specific.
6. **Which attribution fields to store** from `user_visits` — none are mandatory; pick what the product needs (UTMs, click IDs, geo, etc.).
7. **Which quiz `user_properties` to map** into the local profile beyond `email` / `password` (optional).
API base URL: `https://api.web2wave.com/api`  
Auth header on every request: `api_key: <WEB2WAVE_API_KEY>`
Official docs index: https://docs.web2wave.com/llms.txt
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
4. Optionally write local `external_user_id` back to web2wave
5. Set `app_installed=1` **once**
6. Send `"App installed"` event **on every** link visit
7. Create a local session (cookie / JWT — project convention) and redirect into the product
8. Show subscription info (all statuses) in Settings, with Manage links
---
## 2. Security rules (non-negotiable)
- Call web2wave **only from the server** (Route Handler, server action, backend service). Never from the browser with the API key.
- Treat `password` from web2wave `user_properties` as a **bootstrap credential**. Hash it with your normal password hasher before storing. Prefer prompting “set a new password” later.
- Validate that the query value looks like a GUID/UUID before calling the API.
- Strip the query param from the URL after login (redirect) so the GUID is not left in browser history longer than needed.
- Log failures without dumping the API key or raw passwords.
---
## 3. Data model (minimum)
Persist at least:
| Field | Notes |
|-------|--------|
| `id` | Local primary key |
| `email` | From web2wave user properties / response |
| `password_hash` | Nullable if no `password` property |
| `web2wave_user_id` | GUID from the URL — **required for later billing/API calls** |
| `app_installed_reported` | Bool — whether `app_installed=1` was already set |
| `login_count` | Increment on each successful web2wave link visit |
| `last_login_at` / `last_login_ip` / `last_user_agent` | For device/IP context |
| `attribution` (JSON, optional) | Snapshot from latest or first `user_visits` row |
Optional: cache a subscription snapshot for Settings; still refresh from API when rendering Settings.
---
## 4. API endpoints used
| Step | Method | Path | Docs |
|------|--------|------|------|
| Load properties (email, password, …) | `GET` | `/user/properties?user={guid}` | [get_user-properties](https://docs.web2wave.com/reference/get_user-properties) |
| Attribution / visits | `GET` | `/user/user_visits?user={guid}` | [get_user-user-visits](https://docs.web2wave.com/reference/get_user-user-visits) |
| Set property | `POST` | `/user/properties?user={guid}` body `{ "property", "value" }` | [post_user-properties](https://docs.web2wave.com/reference/post_user-properties) |
| Send event | `POST` | `/user/events?user={guid}` | [post_user-events](https://docs.web2wave.com/reference/post_user-events) |
| List subscriptions | `GET` | `/subscriptions/list?user_id={guid}` | [get_subscriptions-list](https://docs.web2wave.com/reference/get_subscriptions-list) |
| (alt) User subscriptions | `GET` | `/user/subscriptions?user={guid}` | [get_user-subscriptions](https://docs.web2wave.com/reference/get_user-subscriptions) |
For Settings UI you need **`manage_link`** and plan name (`price.plan.name`). Prefer `/subscriptions/list` **without** `summary=true` so heavy fields (including `manage_link` and `price`) are present.
---
## 5. Server flow (pseudocode)
```text
ON GET/POST /signin?{QUERY_PARAM}={guid}
  IF missing WEB2WAVE_API_KEY → fail closed (config error)
  IF guid invalid → 400 + friendly error
  props = GET /user/properties?user=guid
  IF 404 / no user → 404 "web2wave user not found"
  email = props.email OR property(props, "email")
  IF no email → 422 "email required from web2wave"
  passwordPlain = property(props, "password")   // may be null
  visits = GET /user/user_visits?user=guid      // best-effort; empty OK
  attribution = pickVisitFields(visits)         // project chooses fields
  local = findByWeb2waveUserId(guid) OR findByEmail(email)
  IF local is null:
      local = createUser({
        email,
        password_hash: passwordPlain ? hash(passwordPlain) : null,
        web2wave_user_id: guid,
        attribution,
        login_count: 0,
        app_installed_reported: false
      })
  ELSE IF local.email == email AND local.web2wave_user_id is null/different:
      APPLY email_conflict_policy:
        autologin | merge → set web2wave_user_id=guid, optionally merge attribution
        error → 409 conflict page
  ELSE IF local.web2wave_user_id == guid:
      // returning user — continue
  ELSE:
      // unexpected identity mismatch — fail closed or policy
  local.login_count += 1
  local.last_login_at = now
  local.last_login_ip = request.ip
  local.last_user_agent = request.userAgent
  save(local)
  // Mirror local id back to web2wave (recommended)
  POST /user/properties?user=guid
    { property: "external_user_id", value: string(local.id) }
  // Conversion flag — ONCE
  IF NOT local.app_installed_reported:
      POST /user/properties?user=guid
        { property: "app_installed", value: "1" }
      local.app_installed_reported = true
      save(local)
  // Analytics event — EVERY link visit
  POST /user/events?user=guid
    {
      event_name: "App installed",
      additional_data: [
        { key: "ip", value: request.ip },
        { key: "user_agent", value: request.userAgent },
        { key: "login_count", value: string(local.login_count) },
        { key: "platform", value: detectPlatform(request.userAgent) },
        { key: "external_user_id", value: string(local.id) },
        // add locale / app_version / device hints as available
      ]
    }
  createSession(local)   // cookie / JWT per project
  redirect(POST_LOGIN_PATH)
```
Notes:
- `app_installed=1` is **idempotent once per local user** (`app_installed_reported`).
- `"App installed"` fires on **every** successful processing of the magic link, including return visits.
- Property / event API failures after local user creation should be logged; decide whether to soft-fail (still log the user in) or hard-fail. Soft-fail is usually better for UX; retry `app_installed` on next visit if the flag is still false.
---
## 6. Settings / profile UI
Show **all** subscriptions returned for the user (not only `active` / `trialing`).
Suggested row copy:
```text
{status}: {product_name}, Until: {until_date}  [Manage]
```
Field mapping:
| UI | API |
|----|-----|
| Status | `subscription.status` (`active`, `trialing`, `canceled`, `past_due`, `paused`, …) |
