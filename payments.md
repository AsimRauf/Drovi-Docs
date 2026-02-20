# Square Payment Integration — Mobile Developer Guide

This document explains how Square credit card payments work in the Drovi backend and how to integrate them into the mobile app (iOS/Android).

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture Flow](#architecture-flow)
3. [Backend API Reference](#backend-api-reference)
4. [Mobile Integration Steps](#mobile-integration-steps)
5. [Customer Order Payment (CARD)](#1-customer-order-payment-card)
6. [Rider Wallet Deposit](#2-rider-wallet-deposit)
7. [Testing with Square Sandbox](#testing-with-square-sandbox)
8. [Square Sandbox Test Cards](#square-sandbox-test-cards)
9. [Testing with Postman / cURL](#testing-with-postman--curl)
10. [Environment Variables](#environment-variables)

---

## Overview

The backend uses the **Square Payments API** (`square` SDK v44). The mobile app's only responsibility is:

1. Initialize the Square Mobile Payments SDK with config from the backend
2. Collect card details → get a **payment token (nonce)** from Square
3. Send that nonce to the backend API

The backend handles charging, confirmation (via webhook), and all business logic.

---

## Architecture Flow

```
┌─────────────┐         ┌──────────────┐         ┌─────────────┐
│  Mobile App │         │ Drovi Backend│         │  Square API │
└──────┬──────┘         └──────┬───────┘         └──────┬──────┘
       │                       │                        │
       │ 1. GET /square/config │                        │
       │──────────────────────>│                        │
       │   { applicationId,    │                        │
       │     locationId,       │                        │
       │     environment }     │                        │
       │<──────────────────────│                        │
       │                       │                        │
       │ 2. Square SDK: show   │                        │
       │    card entry UI      │                        │
       │──────────────────────────────────────────────> │
       │   ← payment nonce     │                        │
       │<──────────────────────────────────────────────  │
       │                       │                        │
       │ 3. POST /orders/      │                        │
       │    confirm-draft OR   │                        │
       │    POST /riders/      │                        │
       │    wallet/deposit     │                        │
       │   { squarePaymentToken│: "cnon:..." }          │
       │──────────────────────>│                        │
       │                       │ 4. payments.create()   │
       │                       │───────────────────────>│
       │                       │   ← paymentId, status  │
       │                       │<───────────────────────│
       │   ← { success, data } │                        │
       │<──────────────────────│                        │
       │                       │                        │
       │                       │ 5. Webhook:            │
       │                       │    payment.completed    │
       │                       │<───────────────────────│
       │                       │ (auto-confirms order   │
       │                       │  or credits wallet)    │
       │                       │                        │
```

---

## Backend API Reference

### Get Square Config (Public — No Auth Required)

```
GET /api/v1/square/config
```

**Response:**
```json
{
  "applicationId": "sandbox-sq0idb-XXXX",
  "locationId": "LXXXXX",
  "environment": "sandbox"
}
```

Use these values to initialize the Square Mobile Payments SDK.

---

## Mobile Integration Steps

### Prerequisites

- **iOS**: Add the [Square Mobile Payments SDK for iOS](https://developer.squareup.com/docs/in-app-payments-sdk/build-on-ios)
- **Android**: Add the [Square Mobile Payments SDK for Android](https://developer.squareup.com/docs/in-app-payments-sdk/build-on-android)
- **React Native**: Use [`react-native-square-in-app-payments`](https://github.com/square/in-app-payments-react-native-plugin)
- **Flutter**: Use [`square_in_app_payments`](https://pub.dev/packages/square_in_app_payments)

### SDK Initialization (Pseudo-code)

```javascript
// 1. Fetch config from backend
const config = await fetch('https://api.drovi.com/api/v1/square/config');
const { applicationId, locationId } = config;

// 2. Initialize Square SDK
SQIPCore.initialize(applicationId);

// 3. Show card entry and get nonce
const cardDetails = await SQIPCardEntry.start(cardEntryConfig);
const nonce = cardDetails.nonce; // e.g. "cnon:card-nonce-ok"

// 4. Send nonce to your backend (see endpoints below)
```

---

## 1. Customer Order Payment (CARD)

When a customer places an order and pays by credit card.

### Endpoint

```
POST /api/v1/orders/confirm-draft
Headers: Authorization: Bearer <customer_jwt>
Content-Type: application/json
```

### Request Body

```json
{
  "draftId": "draft_abc123",
  "paymentMethod": "CARD",
  "squarePaymentToken": "cnon:card-nonce-ok",
  "riderTip": 2.00
}
```

| Field                | Type   | Required | Description                                    |
|----------------------|--------|----------|------------------------------------------------|
| `draftId`            | string | ✅       | The draft order ID                             |
| `paymentMethod`      | string | ✅       | Must be `"CARD"` for credit card payments      |
| `squarePaymentToken` | string | ✅       | The nonce from Square SDK                      |
| `riderTip`           | number | ❌       | Optional tip amount in dollars                 |

### Success Response

```json
{
  "success": true,
  "data": {
    "order": {
      "publicId": "ORD-XXXXXX",
      "status": "PENDING",
      "total": 25.50
    },
    "squarePayment": {
      "paymentId": "sq_pay_XXXX",
      "status": "COMPLETED",
      "receiptUrl": "https://squareup.com/receipt/...",
      "amountCents": 2550
    }
  }
}
```

### What Happens Next

1. Order is created with status `PENDING`
2. Backend charges the card via Square
3. Square sends `payment.completed` webhook → backend auto-confirms order → status becomes `CONFIRMED`
4. Vendor gets notified via WebSocket
5. Customer gets notified via WebSocket

---

## 2. Rider Wallet Deposit

When a rider tops up their wallet balance using a credit card.

### Endpoint

```
POST /api/v1/riders/wallet/deposit
Headers: Authorization: Bearer <rider_jwt>
Content-Type: application/json
```

### Request Body

```json
{
  "amount": 50.00,
  "squarePaymentToken": "cnon:card-nonce-ok"
}
```

| Field                | Type   | Required | Description                              |
|----------------------|--------|----------|------------------------------------------|
| `amount`             | number | ✅       | Deposit amount in **dollars** (min: $1)  |
| `squarePaymentToken` | string | ✅       | The nonce from Square SDK                |

### Success Response

```json
{
  "success": true,
  "message": "Deposit processed",
  "data": {
    "deposit": {
      "id": "dep_XXXX",
      "amount": 50.00,
      "status": "PENDING",
      "squarePaymentId": "sq_pay_XXXX",
      "squareReceiptUrl": "https://squareup.com/receipt/..."
    },
    "squarePayment": {
      "paymentId": "sq_pay_XXXX",
      "status": "COMPLETED",
      "receiptUrl": "https://squareup.com/receipt/...",
      "amountCents": 5000
    }
  }
}
```

### What Happens Next

1. Deposit record created with status `PENDING`
2. Backend charges the card via Square
3. Square sends `payment.completed` webhook → backend credits the rider's wallet
4. Deposit status becomes `COMPLETED`

### Check Wallet Balance

```
GET /api/v1/riders/wallet
Headers: Authorization: Bearer <rider_jwt>
```

### Check Deposit History

```
GET /api/v1/riders/wallet/deposits
Headers: Authorization: Bearer <rider_jwt>
```

---

## Testing with Square Sandbox

### Step 1: Confirm Sandbox Mode

Call `GET /api/v1/square/config` and verify `environment` is `"sandbox"`.

### Step 2: Use Sandbox Nonces (No Real Card Needed)

In **sandbox mode**, Square provides special test nonces you can use directly — no need to show a card entry UI:

| Nonce                          | Simulates                     |
|--------------------------------|-------------------------------|
| `cnon:card-nonce-ok`           | ✅ Successful payment          |
| `cnon:card-nonce-declined`     | ❌ Card declined               |
| `cnon:card-nonce-already-used` | ❌ Nonce already used (error)  |

### Step 3: Test via cURL or Postman

You can test directly without a mobile app using sandbox nonces.

---

## Square Sandbox Test Cards

If you want to test the actual card entry UI in the Square SDK (sandbox mode), use these test cards:

| Card Brand | Number             | CVV  | Expiry   | Zip   | Result    |
|------------|--------------------|------|----------|-------|-----------|
| Visa       | 4532 7153 3985 9856 | 111  | Any future | 12345 | ✅ Success |
| Mastercard | 5166 9263 2349 7532 | 111  | Any future | 12345 | ✅ Success |
| Amex       | 3411 1111 1111 111  | 1111 | Any future | 12345 | ✅ Success |
| Discover   | 6011 0000 0000 0004 | 111  | Any future | 12345 | ✅ Success |
| Declined   | 4000 0000 0000 0002 | 111  | Any future | 12345 | ❌ Declined |

> Full list: https://developer.squareup.com/docs/devtools/sandbox/payments

---

## Testing with Postman / cURL

### Test 1: Order Payment with Card

```bash
# 1. Get Square config
curl https://your-api.com/api/v1/square/config

# 2. Create a draft order first (as customer)
curl -X POST https://your-api.com/api/v1/orders/draft \
  -H "Authorization: Bearer CUSTOMER_JWT" \
  -H "Content-Type: application/json" \
  -d '{
    "restaurantId": "rest_xxx",
    "items": [{ "menuItemId": "item_xxx", "quantity": 1 }],
    "deliveryAddress": { ... }
  }'

# 3. Confirm with card payment (use sandbox nonce)
curl -X POST https://your-api.com/api/v1/orders/confirm-draft \
  -H "Authorization: Bearer CUSTOMER_JWT" \
  -H "Content-Type: application/json" \
  -d '{
    "draftId": "DRAFT_ID_FROM_STEP_2",
    "paymentMethod": "CARD",
    "squarePaymentToken": "cnon:card-nonce-ok"
  }'
```

### Test 2: Rider Wallet Deposit

```bash
# Deposit $50 to rider wallet (use sandbox nonce)
curl -X POST https://your-api.com/api/v1/riders/wallet/deposit \
  -H "Authorization: Bearer RIDER_JWT" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 50,
    "squarePaymentToken": "cnon:card-nonce-ok"
  }'

# Check wallet balance
curl https://your-api.com/api/v1/riders/wallet \
  -H "Authorization: Bearer RIDER_JWT"

# Check deposit history
curl https://your-api.com/api/v1/riders/wallet/deposits \
  -H "Authorization: Bearer RIDER_JWT"
```

### Test 3: Simulate Declined Payment

```bash
curl -X POST https://your-api.com/api/v1/riders/wallet/deposit \
  -H "Authorization: Bearer RIDER_JWT" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 25,
    "squarePaymentToken": "cnon:card-nonce-declined"
  }'
# Expected: 400 Bad Request — "Payment failed: ..."
```

---

## Environment Variables

These must be set on the backend (already configured):

```env
SQUARE_ACCESS_TOKEN=your_square_access_token
SQUARE_APPLICATION_ID=sandbox-sq0idb-XXXX
SQUARE_APPLICATION_SECRET=your_app_secret
SQUARE_LOCATION_ID=LXXXXX
SQUARE_ENVIRONMENT=sandbox          # "sandbox" or "production"
SQUARE_WEBHOOK_SIGNATURE_KEY=...    # Set after creating webhook in Square Dashboard
SQUARE_WEBHOOK_URL=https://your-api.com/api/v1/square/webhooks
```

> **Note:** In sandbox mode, webhook signature verification is skipped if `SQUARE_WEBHOOK_SIGNATURE_KEY` is not set.

---

## Error Handling

| Scenario                | HTTP Status | Error Message                                      |
|-------------------------|-------------|---------------------------------------------------|
| Missing payment token   | 400         | `Square payment token is required for card payments` |
| Card declined           | 400         | `Payment failed: Card declined`                    |
| Nonce already used      | 400         | `Payment failed: Source already used`              |
| Wallet frozen           | 403         | `Your wallet is frozen. Please contact support.`   |
| Rider not found         | 404         | `Rider profile not found`                          |
| Amount too low          | 400         | Validation error (min: $1)                         |

---

## Summary

| Feature             | Endpoint                          | Auth    | Key Payload                                     |
|---------------------|-----------------------------------|---------|-------------------------------------------------|
| Get Square Config   | `GET /api/v1/square/config`       | None    | —                                               |
| Order with Card     | `POST /api/v1/orders/confirm-draft` | Customer | `paymentMethod: "CARD"`, `squarePaymentToken`  |
| Rider Wallet Deposit | `POST /api/v1/riders/wallet/deposit` | Rider  | `amount`, `squarePaymentToken`                  |
| Check Wallet        | `GET /api/v1/riders/wallet`       | Rider   | —                                               |
| Deposit History     | `GET /api/v1/riders/wallet/deposits` | Rider | —                                               |
