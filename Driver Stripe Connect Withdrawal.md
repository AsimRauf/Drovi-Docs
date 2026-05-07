# Drovi Rider Wallet — Stripe Connect Mobile Integration Guide

A stack-agnostic reference for integrating Stripe Connect (Express)
withdrawals into the **rider mobile app**. Written for any framework —
Flutter, React Native, native iOS/Android, etc.

The web rider-wallet page (`drovi-frontend/src/app/dashboard/rider-wallet/page.tsx`)
and the restaurant-mobile app (`drovi-restaurant-mobile/app/(dashboard)/wallet.tsx`)
both already implement this contract end-to-end. Use them as visual /
behavioural references when in doubt — the backend will speak the same
HTTP to your client.

---

## 1. Architecture in one diagram

```
┌──────────────┐   1. GET  /stripe-connect/status?holderType=rider     ┌──────────────────┐
│  Rider App   │   2. POST /stripe-connect/onboard  (returnUrl=…)      │  Drovi Backend   │
│              │ ────────────────────────────────────────────────────► │                  │
│  (Flutter,   │   3. WebView → Stripe Express onboarding URL          │ - status         │
│   RN, etc.)  │   4. Stripe redirects to mobile-bridge HTTPS URL      │ - onboard        │
│              │   5. Bridge HTML auto-redirects to deep link          │ - disconnect     │
│              │   6. WebView closes → app refreshes status            │ - mobile-bridge  │
│              │                                                       │                  │
│              │   7. POST /riders/wallet/withdraw                     │ - withdraw       │
│              │      { amount, payoutSpeed: STANDARD | INSTANT }      │   ↓              │
│              │                                                       │ Stripe Connect   │
│              │   8. DELETE /stripe-connect/disconnect                │   transfer +     │
│              │      ?holderType=rider                                │   (optional)     │
│              │                                                       │   instant payout │
└──────────────┘                                                       └──────────────────┘
```

Money path:
**Customer → Drovi platform Stripe balance → (transfer) → rider's connected
account → (auto payout, T+2 standard / instant) → rider's bank.**
Drovi never holds rider funds outside Stripe; Stripe is the merchant
of record for the connected account.

---

## 2. Endpoint reference

**Base URL (production):** `https://drove-backend-f1d3892431b4.herokuapp.com/api/v1`

All endpoints below are relative to that base. They require the rider's
existing JWT bearer token in `Authorization`, **except** `mobile-bridge`
which is public (Stripe calls it unauthenticated as a redirect target).

If you're testing against staging or a local backend, just substitute
the host — the path shapes (`/stripe-connect/...`, `/riders/wallet/...`)
are identical.

### 2.1 `GET /stripe-connect/status?holderType=rider`

**Purpose:** Read the rider's current Stripe Connect onboarding state.
Call on every wallet-screen mount + every time the app foregrounds
after onboarding.

**Response:**
```json
{
  "success": true,
  "data": {
    "connected": false,                  // Has the rider linked an account at all?
    "chargesEnabled": false,             // Stripe says the account can charge (we never use this for riders)
    "payoutsEnabled": false,             // Required to call /withdraw
    "instantPayoutEligible": false,      // Has an instant-eligible debit card attached
    "detailsSubmitted": false,           // Onboarding form completed
    "needsMoreInfo": false,              // Stripe still wants more data
    "pendingRequirementCount": 0         // Coarse "how much more?" signal
  }
}
```

**Rules:**
- The response **never** contains the raw Stripe account ID or the raw
  `requirements` object. Don't ask for them — they're internal.
- `connected && payoutsEnabled` → the rider can withdraw.
- `connected && !payoutsEnabled` → onboarding started but incomplete;
  CTA should be "Finish onboarding" and re-route through `/onboard` to
  refresh the link.

### 2.2 `POST /stripe-connect/onboard`

**Purpose:** Get a fresh Stripe-hosted onboarding URL. Idempotent on
account creation — first call provisions the Express account, every
subsequent call returns a new short-lived link to the same account.

**Body:**
```json
{
  "holderType": "rider",
  "returnUrl":  "https://drove-backend-f1d3892431b4.herokuapp.com/api/v1/stripe-connect/mobile-bridge?scheme=drovi-rider&path=connect/return",
  "refreshUrl": "https://drove-backend-f1d3892431b4.herokuapp.com/api/v1/stripe-connect/mobile-bridge?scheme=drovi-rider&path=connect/refresh"
}
```

**Response:**
```json
{ "success": true, "data": { "url": "https://connect.stripe.com/express/...", "stripeAccountId": "acct_…" } }
```

**Critical: the bridge URL is non-negotiable.** Stripe rejects custom-
scheme URLs (`drovi-rider://...`) as `return_url` / `refresh_url` and
returns "invalid URL". The platform exposes a public HTTPS bridge that
serves a tiny HTML redirector to your deep link — see §3 below.
**Never** pass a custom-scheme URL directly to `/onboard`.

### 2.3 `GET /stripe-connect/mobile-bridge?scheme=…&path=…` (public)

Returns an HTML page that immediately `window.location.replace`s to
`<scheme>://<path>`. Stripe's hosted onboarding redirects to this URL
when the rider finishes (or hits "Back to app"); the OS then hands the
deep link to your app and the in-app browser auto-closes.

**Allowed `scheme` values** (server-side allowlist):
- `drovi-rider`, `drovi-rider-dev`
- `drovi-restaurant`, `drovi-restaurant-dev`
- `drovi-shop`, `drovi-shop-dev`
- `drovi`, `drovi-dev`

**Allowed `path` values:**
- `connect/return`
- `connect/refresh`

**If you need a different scheme** (e.g. you're shipping a v2 app with
a new bundle ID), ask the backend team to add it to
`drovi-backend/src/stripe/stripe-connect.controller.ts` —
`ALLOWED_MOBILE_SCHEMES`. Anything not on the list returns 400.

### 2.4 `DELETE /stripe-connect/disconnect?holderType=rider`

**Purpose:** Sever the rider ↔ Stripe link locally so they can connect
a different account. Does NOT delete the Stripe account itself.

**Response (success):**
```json
{ "success": true, "data": { "disconnected": true } }
```

**Refused (409 Conflict)** when:
- Any rider withdrawal is in `PENDING` or `PROCESSING` status, or
- The connected account still has a non-zero balance (Stripe will
  auto-payout to the rider's bank later — must let it drain first).

The body's `message` field is already user-safe — render it directly.

### 2.5 `POST /riders/wallet/withdraw`

**Body:**
```json
{ "amount": 25.00, "payoutSpeed": "STANDARD" }
```

`payoutSpeed`: `"STANDARD"` (default, free-ish) or `"INSTANT"` (1.6%
fee, requires `instantPayoutEligible: true`).

**Pre-conditions enforced server-side** (each returns 400 with a
friendly message):
- Rider must be onboarded (`stripeAccountId` set) and `payoutsEnabled`.
- Rider wallet balance ≥ amount.
- Net (after fee) > 0.
- Drovi platform's own Stripe balance covers the transfer (otherwise
  "Payouts are temporarily unavailable. Please try again in a few minutes.").

**Response:** the new `WalletWithdrawal` row. Use `status` (`PENDING`,
`PROCESSING`, `COMPLETED`, `CANCELLED`) for UI; see §6 for what each
means.

### 2.6 `GET /riders/wallet/withdrawals?page=1&limit=20`

Paginated history. Render `failureReason` only after passing it
through the safe-message filter (see §7).

---

## 3. Onboarding flow — how the rider gets back to the app

This is the part that trips up first-time integrators. The mechanism
has FIVE entities involved:

```
┌────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌────────┐
│ Rider  │   │ Your App │   │  Drovi   │   │  Stripe  │   │   OS   │
│        │   │          │   │ Backend  │   │  Hosted  │   │ Router │
└────────┘   └──────────┘   └──────────┘   └──────────┘   └────────┘
```

The trick: Stripe **only accepts HTTPS URLs** for `return_url` /
`refresh_url`. Custom-scheme URLs (`drovi-rider://...`) are rejected
outright with "invalid URL" — that's why a naive integration fails.

So we use a **bridge**: an HTTPS endpoint on Drovi's backend that does
nothing but serve a tiny HTML page which immediately redirects to the
deep link. Stripe is happy (HTTPS), the OS is happy (the deep link
shows up in the in-app browser), and your in-app browser session
recognises its redirect target and auto-closes.

### 3.1 The eight steps, end to end

This is the exact sequence — what each entity does, in order. Use it
as your mental model when debugging.

| # | Who | What |
|---|---|---|
| 1 | **Your App** | Rider taps "Connect Stripe & Withdraw". App constructs `returnUrl` and `refreshUrl` as **HTTPS bridge URLs** pointing at Drovi backend (see §3.2). |
| 2 | **Your App → Drovi Backend** | `POST /stripe-connect/onboard` with `{ holderType: "rider", returnUrl, refreshUrl }`. |
| 3 | **Drovi Backend → Stripe** | Backend creates an Express account (first call only) + asks Stripe for a hosted onboarding link, passing your bridge URLs as `return_url` / `refresh_url`. Returns `{ url, stripeAccountId }` to your app. |
| 4 | **Your App** | Open an **in-app auth session** (NOT a regular WebView, NOT the system browser) with: `openAuthSession(stripeOnboardingUrl, callbackScheme: "drovi-rider")`. The session is told to watch for any URL whose scheme is `drovi-rider://`. Stripe's hosted onboarding loads. |
| 5 | **Rider → Stripe Hosted** | Rider fills SSN, bank account, address, etc. on Stripe's page. Stripe verifies. When they tap "Done" (or hit the back button), Stripe navigates the in-app browser to `returnUrl` (the bridge HTTPS URL). |
| 6 | **Drovi Backend (bridge)** | The bridge endpoint serves an HTML page that contains both `<meta http-equiv="refresh" content="0;url=drovi-rider://connect/return">` and `window.location.replace("drovi-rider://connect/return")`. The browser navigates to that custom-scheme URL. |
| 7 | **OS Router** | The OS sees a `drovi-rider://` URL in the in-app browser. It checks which app registered that scheme — **your app** — and hands the URL over. |
| 8 | **In-app session auto-closes** | Because step 4 told the auth session to watch for `drovi-rider://`, it sees the OS routing happen and **closes itself**. Your app is back in the foreground. The session's completion handler fires with success. |

After step 8, **always** re-fetch `/stripe-connect/status` to know the
real state (don't trust session-result enums — see §3.5).

### 3.2 Building the URLs

The two bridge URLs you pass to `/onboard` must be HTTPS and point at
Drovi's `mobile-bridge` endpoint. The `scheme` query param must match
your app's deep-link scheme (registered in Info.plist / Manifest), and
the `path` must be either `connect/return` or `connect/refresh`.

```
SCHEME      = appDeepLinkScheme()     // "drovi-rider" prod, "drovi-rider-dev" dev
API_BASE    = "https://drove-backend-f1d3892431b4.herokuapp.com/api/v1"

returnUrl   = API_BASE + "/stripe-connect/mobile-bridge"
                       + "?scheme=" + urlEncode(SCHEME)
                       + "&path="   + urlEncode("connect/return")

refreshUrl  = API_BASE + "/stripe-connect/mobile-bridge"
                       + "?scheme=" + urlEncode(SCHEME)
                       + "&path="   + urlEncode("connect/refresh")

callback    = SCHEME + "://"     // what the in-app browser session watches for
```

`appDeepLinkScheme()` should read from your build config — **never
hardcode** — because dev and prod schemes differ
(`drovi-rider-dev` vs `drovi-rider`).

### 3.3 Opening the in-app auth session — by stack

You need your platform's "ASWebAuthenticationSession on iOS, Chrome
Custom Tabs on Android, watching for a redirect URL" primitive — NOT
a normal WebView and NOT a system browser launch. The auto-close on
deep-link is what makes this whole flow work.

| Stack | Code (pseudocode) |
|---|---|
| **Flutter** | `final result = await FlutterWebAuth2.authenticate(url: stripeUrl, callbackUrlScheme: "drovi-rider");` |
| **React Native (Expo)** | `await WebBrowser.openAuthSessionAsync(stripeUrl, "drovi-rider://connect/return");` |
| **React Native (bare)** | `react-native-inappbrowser-reborn` → `InAppBrowser.openAuth(stripeUrl, "drovi-rider://connect/return")` |
| **Native iOS (Swift)** | `ASWebAuthenticationSession(url: stripeUrl, callbackURLScheme: "drovi-rider") { url, error in … }` |
| **Native Android (Kotlin)** | Use Chrome Custom Tabs + a `BroadcastReceiver` keyed off your deep-link `intent-filter`; close the Custom Tab when you receive the deep-link intent. (Or use AppAuth-Android, which wraps this.) |

### 3.4 Registering the deep-link scheme

The OS will only route a `drovi-rider://` URL to your app if your app
declares it.

**iOS (`Info.plist`):**
```xml
<key>CFBundleURLTypes</key>
<array>
  <dict>
    <key>CFBundleURLSchemes</key>
    <array><string>drovi-rider</string></array>
  </dict>
</array>
```

**Android (`AndroidManifest.xml`, inside your launcher `<activity>`):**
```xml
<intent-filter android:autoVerify="false">
  <action android:name="android.intent.action.VIEW"/>
  <category android:name="android.intent.category.DEFAULT"/>
  <category android:name="android.intent.category.BROWSABLE"/>
  <data android:scheme="drovi-rider"/>
</intent-filter>
```

**Flutter** — same as above, plus declare in your platform-specific
config per the `flutter_web_auth_2` README.

**React Native (Expo)** — set `"scheme": "drovi-rider"` (and
`"drovi-rider-dev"` for dev) in `app.json`. `Linking.createURL("")`
will then return `drovi-rider://`.

> **If your scheme isn't already on the backend allowlist** (see §2.3),
> ask the Drovi backend team to add it. Anything not on the list returns
> 400 Bad Request from the bridge endpoint.

### 3.5 What to do after the session closes

The auth session resolves in three ways:

| Result | Cause | What to do |
|---|---|---|
| **success** | Browser saw the `drovi-rider://` URL and closed cleanly | Re-fetch `/stripe-connect/status`, re-render. |
| **cancel** | Rider tapped "Cancel" in the in-app browser | Re-fetch `/stripe-connect/status` anyway — they may have submitted partial info before cancelling, in which case `connected:true` but `payoutsEnabled:false`. |
| **error / timeout** | Network blip, OS killed the browser, etc. | Same — re-fetch `/stripe-connect/status`, re-render. Show a toast if needed. |

**Always re-fetch status after the session closes**, regardless of
result type. Status is the source of truth — the session result is
just a hint.

```
afterAuthSession() async {
  await loadWalletData();   // hits /stripe-connect/status, /riders/wallet, /riders/wallet/withdrawals
  // UI re-renders based on connectStatus.connected / payoutsEnabled
}
```

### 3.6 What if the rider closes the browser before Stripe redirects?

Possible — they tap the back button mid-onboarding, or the OS kills
the browser to free memory. In that case the auth session resolves as
`cancel`, **the deep link is never fired**, and the rider is back in
your app with whatever Stripe had saved at the moment they bailed
(usually nothing — Stripe only persists when they tap "Done").

Your status fetch in §3.5 will tell you exactly where things stand
(probably `connected:false` if they bailed before completion). The CTA
should still read "Connect Stripe & Withdraw" — let them try again.

---

## 4. Withdraw flow

### 4.1 Fee math (mirror server-side)

Implement client-side so the rider sees a live preview. **Server is the
source of truth** — the actual debited amount comes from `/withdraw`'s
response, not from your local computation.

```
function computeFee(amountUsd, speed):
  if amountUsd <= 0: return 0
  if speed == "INSTANT": return round(amountUsd * 0.016, 2)
  return round(amountUsd * 0.003 + 0.30, 2)
```

| Speed | Formula | Drovi's note |
|---|---|---|
| `STANDARD` | 0.30% + $0.30 flat | Stripe's auto-payout schedule (~T+2). |
| `INSTANT` | 1.60% of amount | Lands at bank in minutes. Requires eligible debit card. |

UI: show **Gross / Fee / You receive**.

### 4.2 Submit

```
POST /riders/wallet/withdraw
   body: { amount, payoutSpeed }
→ WalletWithdrawal { id, amount, feeAmount, netAmount, status, ... }
```

Show:
- `INSTANT` success → "Instant payout on its way to your bank."
- `STANDARD` success → "Standard payout created. Stripe will deposit on its regular schedule."

### 4.3 Disable INSTANT when ineligible

```
if !connectStatus.instantPayoutEligible:
  disable INSTANT option, show "Requires an eligible debit card on your Stripe account."
```

The rider adds an eligible debit card via the Stripe Express dashboard,
not via Drovi.

---

## 5. Disconnect flow

```
DELETE /stripe-connect/disconnect?holderType=rider
```

UI:
1. User taps "Disconnect" on the connected status pill.
2. Show a confirmation dialog (destructive style):
   > **Disconnect Stripe?** You will not be able to withdraw funds
   > until you connect a Stripe account again. This will not delete
   > your Stripe account — it just unlinks it from Drovi so you can
   > connect a different one.
3. On confirm → `DELETE`. On 409 (pending withdrawals or non-zero
   connected balance) → render the server's `message` field through
   the safe-message filter.
4. On success → reload wallet data; the "Connect Stripe" CTA reappears.

---

## 6. Withdrawal status lifecycle

| Status | What it means today | UI copy |
|---|---|---|
| `PENDING` | Reserved for legacy admin-approval rows; new Stripe Connect path doesn't use it | "Pending" |
| `PROCESSING` | `STANDARD`: transfer fired to connected account, waiting on bank | "On the way to your bank" |
| `COMPLETED` | `INSTANT`: at the rider's bank. `STANDARD`: transfer to connected account complete (NOT yet at bank) ⚠ | "Deposited" |
| `CANCELLED` | Failed or pre-flight refused; wallet was refunded | "Cancelled — refunded" |

**Known caveat (planned fix on backend):** for `STANDARD`, our system
currently flips to `COMPLETED` on `transfer.created` (transfer to
connected account), NOT on `payout.paid` (transfer to bank). The
auto-payout from connected → bank still happens on Stripe's normal
schedule, but our timestamp is "early". When the backend ships the
`payout.paid` listener, `STANDARD` will stay `PROCESSING` until the
bank actually receives the money. Until then, the safer copy is what's
above (don't tell the user "the money is at your bank" for a STANDARD
that's only just been transferred).

---

## 7. Error handling — defence in depth

The backend already maps Stripe errors to user-safe text via
`toUserSafeMessage`. **Do not** render `error.response.data.message`
raw — implement a thin client-side filter that drops anything looking
like a Stripe internal:

```
function friendlyServerMessage(raw, fallback):
  msg = trim(raw or "")
  if msg is empty: return fallback
  if length(msg) > 200: return fallback
  if matches /(^|[^a-z])(acct|tr|po|cus|pi|ch|in|sub|seti|src|re|txn|py|ba|card)_[A-Za-z0-9]{8,}/: return fallback
  if msg contains any of:
       "balance_insufficient", "insufficient_funds",
       "platform_account_required", "idempotency_error",
       "rate_limit", "api_connection_error", "StripeError"
     : return fallback
  return msg
```

Use it on **every** error path:
- Status fetch
- Onboarding
- Withdraw
- Disconnect
- Historical `failureReason` rendering

Reference implementations:
- Web: `drovi-frontend/src/lib/friendly-server-message.ts`
- Mobile (RN/TS): `drovi-restaurant-mobile/app/(dashboard)/wallet.tsx` →
  `friendlyServerMessage` inside the component

---

## 8. UX rules to follow (don't deviate)

1. **The big action button changes label by state**, not enabled/disabled
   on balance:
   - Not connected → "Connect Stripe & Withdraw"
   - Connected but onboarding incomplete → "Finish Stripe Onboarding"
   - Connected + payouts enabled → "Withdraw"
   - Disable only on `wallet.isFrozen` or while a request is in-flight.
2. **Show the connect status pill** when `connected === true`: a green
   dot + "Stripe payouts enabled", or a yellow dot + "Stripe onboarding
   in progress". Add a small lightning badge if `instantPayoutEligible`.
3. **Always show the withdraw fee summary** before confirm.
4. **After onboarding**, refresh from the server before flipping any
   UI state. Don't infer "connected" from session result.
5. **Never** display `stripeAccountId`, `stripeTransferId`, or
   `stripePayoutId` to the rider. They're not user-facing identifiers.
6. **Frozen wallet** (`wallet.isFrozen === true`) — show the
   `frozenReason` and disable both deposit and withdraw buttons.

---

## 9. Testing

### 9.1 Test mode credentials

The Drovi backend on staging uses Stripe test mode. The drovi backend
team can give you a test rider account already provisioned. To test
end-to-end:

1. Have backend top up the platform's test Stripe balance (uses
   `tok_bypassPending`) — without this, every withdrawal pre-flight
   fails. Ask before testing.
2. Onboard the test rider via `/onboard` → in-app browser opens Stripe
   Express test onboarding. Use Stripe's test SSN `000-00-0000`,
   `000000000` for routing, etc. Stripe pre-fills most fields.
3. After onboarding, `payoutsEnabled` flips to `true` within seconds
   (sometimes on the next `account.updated` webhook).

### 9.2 What to verify before shipping

- [ ] First-time onboarding (no existing account) — full path works
- [ ] Reconnect after Disconnect — second `acct_…` is created cleanly
- [ ] Onboarding cancelled mid-flow — status remains `connected:
      false`, no orphan UI state
- [ ] Withdraw at exact balance, $0.01 over balance, $0 → all give
      friendly errors
- [ ] STANDARD withdraw → row appears, see `PROCESSING` for at least
      a minute, then `COMPLETED`
- [ ] INSTANT withdraw when `instantPayoutEligible: false` → option
      disabled, can't be selected
- [ ] Disconnect refused while a STANDARD is `PROCESSING` — show the
      server's friendly message
- [ ] Network failure during onboarding open — re-tapping retries
      cleanly (no duplicate `acct_…`)
- [ ] Logout / login → wallet rehydrates correctly

### 9.3 Webhook visibility (for ops)

The backend logs raw Stripe errors at WARN/ERROR level with the rider
ID. If you're debugging a failure that the user is reporting, the
backend log is the source of truth — your client only sees the
sanitized version.

---

## 10. Common pitfalls — read before you start

1. **"invalid URL" from Stripe onboarding** → you passed a
   custom-scheme URL directly. Use the bridge.
2. **In-app browser doesn't auto-close after onboarding** → your
   `callbackUrlScheme` / `redirectUrl` doesn't match what your
   manifest declares. Both must be `drovi-rider` (no `://`, no path).
3. **Onboarding succeeds but `payoutsEnabled` stays `false`** →
   `account.updated` webhook may be lagging (or pointed at a different
   environment). Re-fetch `/status` after a 2-second delay; usually
   resolves.
4. **"Payouts are temporarily unavailable"** → not a bug in your
   client. The platform's own Stripe balance is low. Talk to ops.
5. **Disconnect refused with "funds in transit"** → this is correct
   behaviour. Wait for Stripe's auto-payout to settle, then retry.
6. **Two riders share the same connected account** → not possible by
   design. Each `userId` provisions its own `acct_…`. If you see this
   in test data, file a bug.
7. **Showing `stripeAccountId` in the UI** → don't. It's PII-adjacent
   and the response endpoint stopped returning it for a reason.

---

## 11. Endpoint summary cheatsheet

| Method | Path | Purpose |
|---|---|---|
| `GET`  | `/stripe-connect/status?holderType=rider` | Read onboarding state |
| `POST` | `/stripe-connect/onboard` | Get hosted onboarding URL (returnUrl/refreshUrl required) |
| `POST` | `/stripe-connect/refresh-link` | Same as onboard (semantic alias) |
| `DELETE` | `/stripe-connect/disconnect?holderType=rider` | Unlink |
| `GET`  | `/stripe-connect/mobile-bridge?scheme=…&path=…` | Public HTTPS→deep-link redirector |
| `POST` | `/riders/wallet/withdraw` | Submit Stripe Connect withdrawal |
| `GET`  | `/riders/wallet/withdrawals?page&limit` | Paginated history |
| `GET`  | `/riders/wallet` | Wallet balance + transactions |

---

## 12. Reference implementations

When something in this guide is ambiguous, check these — they are
authoritative:

- `drovi-restaurant-mobile/app/(dashboard)/wallet.tsx` — full mobile
  flow (React Native / Expo). The bridge URL construction, error
  filter, and confirm dialog patterns are all here.
- `drovi-frontend/src/app/dashboard/rider-wallet/page.tsx` — web rider
  flow. Same UX, no bridge needed (same-origin URLs work in browser).
- `drovi-backend/src/stripe/stripe-connect.{controller,service}.ts` —
  the actual server contract.

If your client behaves differently from these references, your client
is wrong — not the references.
