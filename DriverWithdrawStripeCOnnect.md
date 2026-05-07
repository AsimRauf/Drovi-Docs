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

Base URL: same `API_BASE` your app already uses (e.g. `https://api.drovi.com/api/v1`).
All endpoints below require the rider's existing JWT bearer token in
`Authorization`, **except** `mobile-bridge` which is public (Stripe
calls it unauthenticated as a redirect target).

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
  "returnUrl":  "https://api.drovi.com/api/v1/stripe-connect/mobile-bridge?scheme=drovi-rider&path=connect/return",
  "refreshUrl": "https://api.drovi.com/api/v1/stripe-connect/mobile-bridge?scheme=drovi-rider&path=connect/refresh"
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

## 3. The mobile onboarding flow, step by step

This is the part that trips up first-time integrators. Read carefully.

### 3.1 What you must build

1. A "Connect Stripe" CTA on the wallet screen, shown when
   `connectStatus.connected === false`.
2. A "Finish Stripe onboarding" CTA when `connected && !payoutsEnabled`.
3. A "Withdraw" CTA when `connected && payoutsEnabled`.

All three CTAs share the same handler at the start: open an in-app
browser session pointed at the Stripe URL. The URL itself comes from
`/stripe-connect/onboard`.

### 3.2 Construct the bridge URLs

Pseudocode:

```
SCHEME = appDeepLinkScheme()        // e.g. "drovi-rider" in prod, "drovi-rider-dev" in dev

bridge(path) =
  API_BASE + "/stripe-connect/mobile-bridge"
  + "?scheme=" + urlEncode(SCHEME)
  + "&path="   + urlEncode(path)

returnUrl  = bridge("connect/return")
refreshUrl = bridge("connect/refresh")

deepLinkReturn = SCHEME + "://connect/return"   // what your in-app browser will watch for
```

`appDeepLinkScheme()` should read the value from your build config —
**not hardcoded** — because dev and prod schemes differ.

### 3.3 Call `/onboard` and open Stripe

```
POST /stripe-connect/onboard
   body: { holderType: "rider", returnUrl, refreshUrl }
→ { url, stripeAccountId }

openInAppAuthSession(url, deepLinkReturn)
```

`openInAppAuthSession` is your platform's "ASWebAuthenticationSession
on iOS, Custom Tabs on Android, watching for a redirect URL" primitive:

| Stack | Function |
|---|---|
| Flutter | `flutter_web_auth_2: authenticate(url, callbackUrlScheme: "drovi-rider")` |
| React Native (Expo) | `WebBrowser.openAuthSessionAsync(url, deepLinkReturn)` |
| Native iOS | `ASWebAuthenticationSession(url:, callbackURLScheme:, completionHandler:)` |
| Native Android | Custom Tabs + an intent filter on the deep-link scheme |

The session **auto-closes** when the in-app browser navigates to
`deepLinkReturn`. Stripe's onboarding redirects to your bridge HTTPS
URL → bridge HTML redirects to `drovi-rider://connect/return` → OS
recognises the scheme → session closes with success.

### 3.4 After the session closes

Either way (success, cancel, or timeout) — re-fetch `/stripe-connect/status`
and re-render. Don't trust the session result type for state; trust
the status endpoint.

```
afterSession() {
  loadWalletData()   // hits /status, /wallet, /withdrawals
}
```

### 3.5 Configure the deep link

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

**Android (`AndroidManifest.xml`, in your launcher activity):**
```xml
<intent-filter>
  <action android:name="android.intent.action.VIEW"/>
  <category android:name="android.intent.category.DEFAULT"/>
  <category android:name="android.intent.category.BROWSABLE"/>
  <data android:scheme="drovi-rider"/>
</intent-filter>
```

**Flutter** — same plus declare in `pubspec.yaml`/`Info.plist` per
`flutter_web_auth_2` README.

**React Native (Expo)** — set `"scheme"` in `app.json`. `Linking.createURL("")`
returns `<scheme>://` so you don't need to hardcode.

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
