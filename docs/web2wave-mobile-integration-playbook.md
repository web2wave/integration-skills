# web2wave Mobile App Integration Playbook

Framework-agnostic instructions for AI agents and developers connecting a **mobile app** (iOS, Android, Flutter, React Native, Unity) to **web2wave**: resolve the web2wave user after install, give them access to what they paid for on web, and (optionally) let them manage the subscription.

Copy this file into your agent context (Cursor skill, Claude project doc, Codex instructions, etc.) and follow it end-to-end.

Code samples are **not duplicated here**. Every step links to the maintained documentation and SDK repositories — read them and adapt the sample to the project's language. Doc index for agents: https://docs.web2wave.com/llms.txt

```text
API base:  https://api.web2wave.com/api
Auth:      header api-key: <WEB2WAVE_API_KEY>      (not Bearer; `api_key` is accepted as an alias)
Docs:      https://docs.web2wave.com/llms.txt
SDKs:      Swift · Kotlin · Java · Flutter · React Native · Unity  (github.com/web2wave/web2wave_*)
```

---

## 0. How to work

1. **Discover first, ask second.** Inspect the project (§1), show the human what you found, and ask only what you cannot determine.
2. **Branch by what already exists** (§2–§4). Do not add a second subscription provider or a second attribution tool.
3. **Prefer the simplest setup.** Recommend no backend (§5) and the standard deeplink format unless the human has a reason.
4. **Do not invent secrets or settings.** If a step depends on web2wave project settings you cannot see, tell the human exactly what to set and where.

---

## 1. Discover the project

Search the codebase before asking anything.

| Look for | Files / identifiers | Meaning |
|---|---|---|
| Platform | `*.xcodeproj`, `Package.swift`, `Podfile`, `build.gradle(.kts)`, `pubspec.yaml`, `package.json` (react-native / expo), Unity `Packages/manifest.json` | Which SDK to use (§7) |
| web2wave SDK | `web2wave`, `Web2Wave` | Already integrated? |
| **RevenueCat** | `purchases`, `RevenueCat`, `Purchases.configure`, `purchases_flutter`, `react-native-purchases` | §3.1 |
| **Adapty** | `Adapty`, `adapty`, `adapty_flutter` | §3.2 |
| **Qonversion** | `Qonversion`, `qonversion_flutter` | §3.2 |
| **Apphud** | `ApphudSDK`, `Apphud`, `apphud` | §3.2 |
| **Superwall** | `SuperwallKit`, `Superwall`, `superwallkit_flutter` | §3.2 |
| **AppsFlyer** | `AppsFlyerLib`, `appsflyer_sdk`, `react-native-appsflyer` | §2 |
| **Adjust** | `Adjust`, `adjust_sdk` | §2 |
| **Branch** | `Branch`, `flutter_branch_sdk` | §2 |
| Existing paywall / StoreKit / Play Billing code | `StoreKit`, `BillingClient` | Native purchases may coexist; do not remove them |

Report to the human: platform, subscription provider (or none), attribution tool (or none). Ask them to correct anything wrong.

If **several** subscription providers are present, ask which one is the source of truth for access in the app.

---

## 2. Get the web2wave `user_id` into the app

The app needs the web2wave user id (a GUID) to talk to web2wave. Pick **one** path.

### 2.1 Attribution tool present (AppsFlyer / Adjust / Branch)

Use its **deferred deeplink**: the web2wave "after pay" link opens the store, and after install the tool hands the `user_id` to the app.

Ask the human which format they use:

1. **Standard (recommended)** — JSON in `deep_link_value` containing `user_id` (AppsFlyer), or `user_id` as a query parameter in the resolved URL (Adjust).
2. **Custom** — they already have their own parameters. Read their format, find `user_id` in it, and make sure the web2wave deeplink helper is configured to produce it.

Documentation (read before coding):

- End-to-end flow: https://docs.web2wave.com/reference/pass-subscription-from-web2web-to-app
- AppsFlyer and Adjust parsing samples, and the exact link formats: https://docs.web2wave.com/reference/revenuecat-web2wave-integration (the samples apply to every provider; replace the provider call)
- The link is built in the web2wave project: **Project settings → Deeplinks & Billing → Helper**.

Also ask: *Do you want the web2wave deferred deeplinks SDK as a fallback* when the MMP does not deliver the `user_id` (see §2.4)?

### 2.2 No attribution tool

**Propose web2wave deferred deeplinks right away** — no extra SDK and no MMP dashboard. Call `identify()` from the web2wave SDK on first launch.

- https://docs.web2wave.com/reference/web2wave-deferred-deeplinks

### 2.3 App opened its own WebView with the quiz / paywall

If the app embeds the quiz or paywall, the user id is known from the WebView session and the provider profile id can be passed in the URL (`…?revenuecat_profile_id=…`, `adapty_profile_id`, `qonversion_profile_id`, `superwall_profile_id`, `apphud_profile_id`).

- https://docs.web2wave.com/reference/embedding-quizzes-and-paywalls-into-mobile-apps

**Store the web2wave `user_id` on the device** (secure storage / keychain / shared prefs). It is needed for access checks and Manage Subscription later.

### 2.4 When the `user_id` does not arrive

Deeplinks fail in practice (the user opened the store link in another browser, reinstalled, or the MMP did not match). Plan for it:

1. **Ask** (if an MMP is present) or **recommend** (if there is none): *use the web2wave deferred deeplinks SDK as the fallback?* On first launch, if no `user_id` was stored and the MMP gave none, call `identify()`.
2. Know its limits: it matches the device to a recent quiz / paywall session, **only within the last 48 hours**. The response `match_method` can be `no_match` (no candidate) or `ambiguous_fingerprint` (several users share the fingerprint) — in both cases there is **no `user_id`**. Handle that state; never crash or loop.
3. When there is still no `user_id`, show a "Restore access" path instead of the paywall. If the human wants lookup by email, require **verification** (for example a login link sent to that email) before granting access — an email address alone is not proof of ownership.
4. For a custom deeplink without an MMP, the usual instruction is "install the app via the link and then **open the link again**".

Reference: https://docs.web2wave.com/reference/web2wave-deferred-deeplinks

### 2.5 After the `user_id` is resolved: skip onboarding and paywall

The user already went through the web funnel and paid. Once the `user_id` is resolved and access is confirmed:

1. **Do not repeat onboarding.** Skip the in-app quiz or funnel. If the app wants to show or reuse the answers, read them from web2wave (`GET /user/properties`, §6.2) and pre-fill the profile.
2. **Do not show the paywall** while the subscription is `active` or `trialing`.
3. Show a short **"Subscription confirmed"** screen, then move the user to the main part of the app (home / dashboard) automatically.
4. If access is **not** confirmed yet (the webhook / sync may take a moment), retry briefly before falling back to the normal paywall.

Reference: https://docs.web2wave.com/reference/post-subscription-integration-flow

---

## 3. Branch by subscription provider

web2wave handles the payment on web. The provider below is only where **access** lives for the app.

### 3.1 RevenueCat

There are two ways to connect. **Explain both to the human before choosing.**

**A. RevenueCat's own Stripe integration (direct).**
RevenueCat reads purchases straight from Stripe. It works, but the project is **tied to Stripe**: they cannot later move web payments to another provider and cannot run payment-provider A/B tests in web2wave. Choose this only if they accept that.

**B. Integration through web2wave (recommended).**
web2wave grants and revokes the RevenueCat entitlement itself. The payment provider stays swappable. Docs: https://docs.web2wave.com/reference/revenuecat-web2wave-integration

For **B**, the single question is *which RevenueCat user receives the entitlement*. In the web2wave project this is controlled by **Create user on subscription** and **ID for RevenueCat** (`Email or user_id` / `user_id` / `app_profile_id`). It works reliably in these setups only:

| Setup | What the app does | web2wave project setting |
|---|---|---|
| **1. App uses the web2wave `user_id` as RevenueCat App User ID** | Passes `user_id` through the deeplink and calls `Purchases.configure(… appUserID: user_id)` / `logIn(user_id)` | ID for RevenueCat = `user_id` |
| **2. App creates RevenueCat users by email** | Calls `logIn(email)` with the same email the user typed in the funnel | ID for RevenueCat = `Email or user_id` |
| **3. Direct Stripe integration, identified by email** | RevenueCat users come from Stripe by email | n/a (Option A) |
| **4. App sends its own id** | Reads RevenueCat `appUserID` and sends it to web2wave (`setRevenuecatProfileID` / `revenuecat_profile_id`) | Create user on subscription = **off**; web2wave waits for the profile id |

Setup 4 works with any RevenueCat configuration, including anonymous ids. It is the safest choice when the app's user identity is unclear.

**Stop and consult web2wave** if none of the above matches the project (for example, users are created by an internal account id, or both web and app already have separate RevenueCat users). Do not guess a mapping — tell the human to ask web2wave support, and wait for the agreed scheme.

Entitlement name: ask for the RevenueCat entitlement identifier. It is set per project and can be overridden per price (and per Android / iOS).

### 3.2 Adapty, Qonversion, Apphud, Superwall

These do **not** use "create user on subscription". The flow is the same for all four:

1. The app resolves the web2wave `user_id` (§2).
2. The app reads the provider's own user/profile id.
3. The app sends it to web2wave with the SDK method (or `POST /user/properties`). As soon as the property is saved, web2wave grants access to that provider user and keeps it in sync (renewal, cancel, refund).

| Provider | Property sent to web2wave | SDK method | Docs |
|---|---|---|---|
| Adapty | `adapty_profile_id` | `setAdaptyProfileID` | https://docs.web2wave.com/reference/adapty-web2wave-integration |
| Qonversion | `qonversion_profile_id` | `setQonversionProfileID` | https://docs.web2wave.com/reference/qonversion-web2wave-integration |
| Apphud | `apphud_profile_id` | `setApphudProfileID` | https://docs.web2wave.com/reference/apphud-web2wave-integration |
| Superwall | `superwall_profile_id` | `setSuperwallProfileID` | https://docs.web2wave.com/reference/superwall-web2wave-integration |

Notes for the agent:

- web2wave **does not create users** in these providers. The provider SDK must already be started in the app so the user exists.
- Superwall: send the id **after** `identify()` so it is the same id Superwall uses.
- Each provider page lists project settings the human must fill (API key, entitlement / product / access level). Show them the page and the field names.
- Android and iOS may need different entitlements or products — see §3.4.

### 3.3 No subscription provider

Check access straight from web2wave. Use the SDK `hasActiveSubscription(userId)` (or the API in §6) on launch and after returning from background. Treat `active` and `trialing` as entitled.

- https://docs.web2wave.com/reference/direct-web2wave-integration

### 3.4 Android and iOS products

Ask: *Are the entitlements / products the same on Android and iOS?*

- **Same:** fill the project-level value (entitlement / product / access level) once.
- **Different:** fill the price-level fields. Price settings have separate **Android** and **iOS** fields (for example RevenueCat and Superwall entitlements, Apphud product ID); see the provider page for the exact field names. Priority is *platform field of the price → price field → project setting*.

web2wave picks the field from the user's `user_platform` property (`android` / `ios`). If it is missing, the generic value is used — so check that it is set for the test user.

---

## 4. Verify the connection

Before moving on, ask the human to run one real or test purchase on web and confirm:

1. The app receives the `user_id` after install (log it).
2. The provider id (or `user_id` for RevenueCat setups 1–2) reaches web2wave: `GET /user/properties?user={guid}` shows the property.
3. The user is entitled in the provider dashboard and in the app.
4. Cancelling the subscription in web2wave removes access.

See acceptance tests in §11.

---

## 5. Backend (optional — recommend "no")

Ask: *Do you also need to know about subscription changes on your own backend?*

Recommend **no** unless they already have server-side logic that depends on it. The app plus the provider is enough, and every extra moving part is another place to break.

If **yes**, use webhooks:

- Webhook formats: https://docs.web2wave.com/reference/webhook-formats
- Setup: **Project settings → API & Webhook → Add webhook**, enable **Subscription updated** (see https://docs.web2wave.com/reference/get-your-api-key-and-set-up-webhooks). Use a public HTTPS URL.

Explain what arrives: a `type: "subscription"` webhook whenever a subscription is created, **renewed**, canceled, or its status changes. The payload includes `status` and `manage_link`. The backend must handle repeats idempotently and respond quickly. Keep the web2wave API key on the server only.

---

## 6. web2wave API contract (used by §2.5, §3.3, §8)

### 6.1 Subscriptions

Verified field names (same as the web playbook):

- `GET /user/subscriptions?user={guid}` → rows in `subscription`.
- `GET /subscriptions/list?user_id={guid}` → paginated, rows in `data` (preferred).
- Not-found often returns **422** `{"error":1,"error_msg":"user not found"}`. Treat 404 and 422 as "user not found", and `error === 1` as failure even on HTTP 200.

Useful fields: `status`, `paywall_name`, `plan_name`, `next_charge_date`, `canceled_at`, `manage_link`, `payment_system_label`.

Entitlement: `status ∈ {active, trialing}` is access. Still list canceled / past_due / paused in any UI.

### 6.2 User properties and quiz answers

`GET /user/properties?user={guid}` →

```json
{
  "user_id": "00000000-0000-4000-8000-000000000001",
  "user_email": "user@example.com",
  "properties": [
    { "property": "email", "value": "user@example.com" },
    { "property": "last_quiz_id", "value": "10001" },
    { "property": "app_installed", "value": "1" }
  ]
}
```

- Email = `user_email` or property `email` (there is usually no top-level `email`).
- Quiz answers and registration data come back as properties. Read only the ones the app needs, and use them to personalize the app and pre-fill the profile (§2.5).

### 6.3 Writing properties and events from the app

Send these after the `user_id` is resolved (the SDKs wrap them):

- Property `app_installed = "1"` — once per user. Optionally also `external_user_id` = the app's own user id.
- Event `"App installed"` — once per install:

```http
POST /user/events?user={guid}
api-key: <key>

{
  "event_name": "App installed",
  "event_value": "1",
  "quiz": "10001",
  "user_agent": "…",
  "additional_data": [ { "key": "platform", "value": "ios" } ]
}
```

- **`event_name`, `event_value` and `quiz` are all required.** Missing `event_value` → 422 "The event value field is required"; missing `quiz` → 422 "The quiz field is required".
- `quiz` ← property `last_quiz_id` (fallback `"app"`).
- `additional_data` is an array of `{key, value}` objects, not a free-form object.

---

## 7. SDK

Use the matching SDK instead of raw HTTP in the app. Add the dependency and read the README of the repository:

| Platform | Repository |
|---|---|
| iOS (Swift) | https://github.com/web2wave/web2wave_swift |
| Android (Kotlin) | https://github.com/web2wave/web2wave_kotlin |
| Android (Java) | https://github.com/web2wave/web2wave_java |
| Flutter | https://github.com/web2wave/web2wave_flutter |
| React Native | https://github.com/web2wave/web2wave_react_native |
| Unity | https://github.com/web2wave/web2wave_unity |

Overview (all SDKs, including React Native): https://docs.web2wave.com/reference/sdk-integration

Method names differ slightly per language (`setApphudProfileID` / `SetApphudProfileID` in Unity). Use the README, not memory.

**Security.** The SDKs authenticate with the project API key, so it ends up inside the app. Treat it as exposed: use it for the read and identify calls the SDKs make, and do **not** call destructive endpoints (cancel, refund, charge) from the app. Those belong on a backend you control.

---

## 8. Manage Subscription button (optional)

Ask: *Do you want a "Manage subscription" button in the app, like on the web?*

If yes:

1. Persist the web2wave `user_id` on first launch (§2).
2. On tap, fetch the user's subscriptions (§6).
3. Take `manage_link` from the subscription row. It **already contains the `user_id`** — do not append it yourself.
4. Open it in the system browser or an in-app browser. Do **not** try to render it inside the native UI.
5. **Hide the button** when no subscription has a non-empty `manage_link` (some payment providers return `""`).
6. Refresh state when the user returns to the app (the plan may have changed).

Put the button where users expect billing: **Settings → Subscription**.

**What the human must set up in web2wave** (tell them exactly where):

1. The self-service page is a **quiz**. Find the **Manage subscriptions** quiz in *Quizzes & Pages*, or create it from the template of the same name, and publish it. It can be edited like any quiz — downsell and cancellation flows are configured there.
2. Save its URL in **Project → General → Customer portal link**, including the `{user_id}` placeholder: `https://quiz.yourdomain.com/manage-subscriptions?user_id={user_id}`. If the field is empty, web2wave falls back to the **Stripe customer portal link** from payment settings.
3. **Fallback without a `user_id`:** the user can sign in by email at `https://YOUR-DOMAIN/manage-account` and receives a magic link to the portal. Offer it as "I can't find my subscription" in the app.

Reference: https://docs.web2wave.com/reference/managing-subscription

---

## 9. Optional: onboarding quizzes inside the app

Ask: *Do you want to use web2wave quizzes for onboarding inside the app?*

If yes, show the quiz in a **WebView** instead of building native onboarding screens. The quiz is edited in web2wave, so onboarding can change without an app release.

1. Embed the quiz URL in the SDK's WebView component (the SDKs ship one — see §7).
2. Pass the provider profile id in the URL when the app already has one (`…?revenuecat_profile_id=…`, `adapty_profile_id`, `qonversion_profile_id`, `superwall_profile_id`, `apphud_profile_id`), so a purchase made in the flow syncs to the right provider user.
3. Handle the WebView events (quiz finished, close, errors) with the SDK listener and continue the native flow.
4. Reuse the same web2wave `user_id` for these users (§2.3) so answers, attribution and subscriptions stay in one profile.

Docs and per-platform samples: https://docs.web2wave.com/reference/embedding-quizzes-and-paywalls-into-mobile-apps

---

## 10. Optional: payment inside the app with web2wave

Ask: *Do you want to accept payments inside the app through web2wave?*

If yes, create a **paywall** in web2wave and open it in the WebView (§9). Payment goes through the payment systems connected to web2wave and syncs directly to the subscription provider (§3), so access appears in the app the same way as for web purchases.

**Tell the human plainly before implementing:**

- This is **officially allowed only in the US**. In other countries, app store rules can make it a problem for the app (review rejection or other consequences). Check the current store policies for the markets they target.
- Outside the US we recommend using it **only for promotions** — for example paywalls linked from email campaigns or push notifications — and **only for users who originally came from the web funnel**, not as the default purchase path for everyone.
- If they are unsure, keep the in-app purchase flow as is and skip this step.

Implementation notes:

1. Create the paywall in web2wave and take its URL.
2. Open it in the SDK WebView with the web2wave `user_id` and the provider profile id in the URL (§9, step 2).
3. Gate the entry point (country check and "came from web" flag) so the paywall is shown only to the intended users.
4. After purchase, re-check access as in §3 / §6.

---

## 11. Acceptance tests

1. Fresh install via the deeplink: the app logs the web2wave `user_id`, and it is stored on the device.
2. Without an attribution tool: `identify()` returns the same `user_id`.
3. **RevenueCat** (per chosen setup): the entitlement appears on the expected RevenueCat user and the app unlocks. Cancel on web → entitlement removed.
4. **Adapty / Qonversion / Apphud / Superwall:** after the profile id is sent, `GET /user/properties` shows it, the provider dashboard shows the access, the app unlocks; cancel → access removed.
5. No provider: `hasActiveSubscription` is true for `active` / `trialing`, false after cancellation.
6. Reinstall / second device with the same deeplink: no duplicate users, access still correct.
7. Unknown or malformed `user_id` is handled (no crash, user sees a sensible message).
8. If backend webhooks are enabled: renewal and cancellation reach the backend once each.
9. Manage Subscription (if enabled): opens the right link, hidden when no link.
10. The API key is not logged and no destructive endpoint is called from the app.
11. Onboarding quiz (if enabled): opens in the WebView, completion returns control to the native flow, answers appear in the user's properties.
12. In-app paywall (if enabled): shown only to the intended users, a test purchase grants access in the provider, and the US-only note was explained to the human.
13. After a successful deeplink the app skips onboarding and the paywall, shows "Subscription confirmed", and opens the main screen.
14. With no `user_id` (`no_match` / `ambiguous_fingerprint`) the app shows "Restore access" and does not crash or show a paywall to a paying user.
15. `app_installed=1` is set once and the `"App installed"` event is accepted (HTTP 200, with `event_value` and `quiz`).
16. If Android and iOS use different products, a test user on each platform receives the right one.

---

## 12. Agent output expectations

When applying this playbook, the agent should:

1. Report the discovery results (§1) and get confirmation.
2. Explain the relevant branch to the human — for RevenueCat, **both** connection ways and the working setups in §3.1.
3. Implement only the chosen branch, using SDK calls and the linked docs for code samples.
4. List, for the human, the exact web2wave project settings to fill (deeplink helper, provider keys, entitlement / product, and the webhook URL under API & Webhook if a backend is needed).
5. Handle the post-deeplink flow (§2.5) and the missing-`user_id` fallback (§2.4), and ask about Android / iOS products (§3.4).
6. Add Manage Subscription only if requested (§8).
7. At the end, ask about in-app onboarding quizzes (§9) and in-app payment (§10), and implement only what the human chooses.
8. Describe or run the acceptance tests (§11), and say which ones could not be run.
