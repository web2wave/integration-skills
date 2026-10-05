# web2wave Mobile App Integration Playbook

Framework-agnostic instructions for AI agents and developers connecting a **mobile app** (iOS, Android, Flutter, React Native, Unity) to **web2wave**: resolve the web2wave user after install, give them access to what they paid for on web, and (optionally) let them manage the subscription.

Copy this file into your agent context (Cursor skill, Claude project doc, Codex instructions, etc.) and follow it end-to-end.

Code samples are **not duplicated here**. Every step links to the maintained documentation and SDK repositories — read them and adapt the sample to the project's language. Doc index for agents: https://docs.web2wave.com/llms.txt

```text
API base:  https://api.web2wave.com/api
Auth:      header api_key: <WEB2WAVE_API_KEY>      (not Bearer)
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
- The link is built in the web2wave project: **Project settings → Deeplinks → Helper**.

### 2.2 No attribution tool

Use web2wave deferred deeplinks: call `identify()` from the SDK on first launch.

- https://docs.web2wave.com/reference/web2wave-deferred-deeplinks

### 2.3 App opened its own WebView with the quiz / paywall

If the app embeds the quiz or paywall, the user id is known from the WebView session and the provider profile id can be passed in the URL (`…?revenuecat_profile_id=…`, `adapty_profile_id`, `qonversion_profile_id`, `superwall_profile_id`, `apphud_profile_id`).

- https://docs.web2wave.com/reference/embedding-quizzes-and-paywalls-into-mobile-apps

**Store the web2wave `user_id` on the device** (secure storage / keychain / shared prefs). It is needed for access checks and Manage Subscription later.

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
- Android / iOS can have different entitlements or products per price — mention it.

### 3.3 No subscription provider

Check access straight from web2wave. Use the SDK `hasActiveSubscription(userId)` (or the API in §6) on launch and after returning from background. Treat `active` and `trialing` as entitled.

- https://docs.web2wave.com/reference/direct-web2wave-integration

---

## 4. Verify the connection

Before moving on, ask the human to run one real or test purchase on web and confirm:

1. The app receives the `user_id` after install (log it).
2. The provider id (or `user_id` for RevenueCat setups 1–2) reaches web2wave: `GET /user/properties?user={guid}` shows the property.
3. The user is entitled in the provider dashboard and in the app.
4. Cancelling the subscription in web2wave removes access.

See acceptance tests in §9.

---

## 5. Backend (optional — recommend "no")

Ask: *Do you also need to know about subscription changes on your own backend?*

Recommend **no** unless they already have server-side logic that depends on it. The app plus the provider is enough, and every extra moving part is another place to break.

If **yes**, use webhooks:

- Webhook formats: https://docs.web2wave.com/reference/webhook-formats
- Setup: **Cabinet → API & Webhooks → Add webhook**, enable **Subscription updated** (see https://docs.web2wave.com/reference/get-your-api-key-and-set-up-webhooks). Use a public HTTPS URL.

Explain what arrives: a `type: "subscription"` webhook whenever a subscription is created, **renewed**, canceled, or its status changes. The payload includes `status` and `manage_link`. The backend must handle repeats idempotently and respond quickly. Keep the web2wave API key on the server only.

---

## 6. Subscription API contract (used by §3.3 and §8)

Verified field names (same as the web playbook):

- `GET /user/subscriptions?user={guid}` → rows in `subscription`.
- `GET /subscriptions/list?user_id={guid}` → paginated, rows in `data` (preferred).
- Not-found often returns **422** `{"error":1,"error_msg":"user not found"}`. Treat 404 and 422 as "user not found", and `error === 1` as failure even on HTTP 200.

Useful fields: `status`, `paywall_name`, `plan_name`, `next_charge_date`, `canceled_at`, `manage_link`, `payment_system_label`.

Entitlement: `status ∈ {active, trialing}` is access. Still list canceled / past_due / paused in any UI.

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

Overview: https://docs.web2wave.com/reference/sdk-integration · React Native: https://docs.web2wave.com/reference/react-native-integration

Method names differ slightly per language (`setApphudProfileID` / `SetApphudProfileID` in Unity). Use the README, not memory.

**Security.** The SDKs authenticate with the project API key, so it ends up inside the app. Treat it as exposed: use it for the read and identify calls the SDKs make, and do **not** call destructive endpoints (cancel, refund, charge) from the app. Those belong on a backend you control.

---

## 8. Manage Subscription button (optional)

Ask: *Do you want a "Manage subscription" button in the app, like on the web?*

If yes:

1. Persist the web2wave `user_id` on first launch (§2).
2. On tap, fetch the user's subscriptions (§6).
3. Take `manage_link`:
   - It can be **per subscription** (taken from each subscription row), or
   - **static** for the whole project (the customer portal link configured in web2wave).
   Prefer the per-subscription value; fall back to the project link.
4. Open it in the system browser or an in-app browser. Do **not** try to render it inside the native UI.
5. **Hide the button** when no subscription has a non-empty `manage_link` (some payment providers return `""`).
6. Refresh state when the user returns to the app (the plan may have changed).

Reference for the web equivalent: https://docs.web2wave.com/reference/managing-subscription

---

## 9. Acceptance tests

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

---

## 10. Agent output expectations

When applying this playbook, the agent should:

1. Report the discovery results (§1) and get confirmation.
2. Explain the relevant branch to the human — for RevenueCat, **both** connection ways and the working setups in §3.1.
3. Implement only the chosen branch, using SDK calls and the linked docs for code samples.
4. List, for the human, the exact web2wave project settings to fill (deeplink helper, provider keys, entitlement / product, and the webhook URL under Cabinet → API & Webhooks if a backend is needed).
5. Add Manage Subscription only if requested (§8).
6. Describe or run the acceptance tests (§9), and say which ones could not be run.
