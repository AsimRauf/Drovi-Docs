# Restaurant Rating & Review System — Mobile Developer Guide

This document explains the rating and review system for restaurant orders in the Drovi app.

---

## Table of Contents

1. [Overview](#overview)
2. [How It Works](#how-it-works)
3. [API Reference](#api-reference)
4. [Review Reminder Flow](#review-reminder-flow)
5. [Restaurant Detail Page — Live Ratings](#restaurant-detail-page--live-ratings)
6. [Testing with cURL / Postman](#testing-with-curl--postman)
7. [Error Handling](#error-handling)

---

## Overview

- Customers can **rate and review delivered restaurant orders** (1–5 stars, optional text, optional photo)
- **One review per order** — enforced by the backend
- Reviews are shown on the **restaurant detail page** (reviewer name, avatar, stars, comment, photo, date)
- **No order info is exposed** on the public review — purely ratings/reviews
- The backend tracks which orders are **pending review**
- A **smart reminder system** nudges the customer after every ~6 app visits (not annoying)

---

## How It Works

```
CUSTOMER PLACES ORDER → ORDER DELIVERED → ORDER BECOMES "PENDING REVIEW"
                                                │
                          ┌─────────────────────┼──────────────────────┐
                          │                     │                      │
                    Orders Page             Reminder System       Restaurant Page
                    shows "Rate Order"      prompts after         shows live
                    button on delivered      6 app visits          rating + reviews
                    unreviewed orders        (1 order, 1 time)     from all customers
                          │                     │                      │
                          └──────────┬──────────┘                      │
                                     │                                 │
                              Customer submits                   Reviews appear
                              POST /reviews                      GET /reviews/restaurant/:id
                              (rating + comment + photo)
```

---

## API Reference

### 1. Submit a Review

```
POST /api/v1/reviews
Headers: Authorization: Bearer <customer_jwt>
Content-Type: application/json
```

**Request Body:**

```json
{
  "orderPublicId": "ORD-ABC123",
  "rating": 5,
  "comment": "Amazing food, fast delivery!",
  "imageUrl": "https://storage.googleapis.com/drovi-bucket/review-images/abc.jpg"
}
```

| Field           | Type   | Required | Description                                 |
|-----------------|--------|----------|---------------------------------------------|
| `orderPublicId` | string | ✅       | Public ID of the delivered order             |
| `rating`        | int    | ✅       | 1 to 5 (integer only)                       |
| `comment`       | string | ❌       | Review text                                  |
| `imageUrl`      | string | ❌       | URL from the image upload endpoint           |

**Success Response:**

```json
{
  "success": true,
  "data": {
    "id": "...",
    "orderId": "...",
    "orderPublicId": "ORD-ABC123",
    "restaurantId": "...",
    "rating": 5,
    "comment": "Amazing food, fast delivery!",
    "imageUrl": "https://...",
    "reviewerName": "John Doe",
    "reviewerAvatar": "https://...",
    "status": "PUBLISHED",
    "createdAt": "2026-02-21T..."
  }
}
```

**Rules enforced by backend:**
- Order must belong to the current user
- Order must have status `DELIVERED`
- Order must be a restaurant order (not shop)
- Only one review per order (returns `409 Conflict` if already reviewed)

---

### 2. Upload Review Image (Optional)

Upload an image **before** submitting the review, then pass the returned URL in `imageUrl`.

```
POST /api/v1/reviews/upload-image
Headers: Authorization: Bearer <customer_jwt>
Content-Type: multipart/form-data

Body: file=<image file>
```

**Response:**

```json
{
  "url": "https://storage.googleapis.com/drovi-bucket/review-images/abc123.jpg"
}
```

---

### 3. Get Pending Review Orders

Returns delivered orders from the **last 90 days** that have NOT been reviewed yet.

```
GET /api/v1/reviews/me/pending-orders
Headers: Authorization: Bearer <customer_jwt>
```

**Response:**

```json
{
  "success": true,
  "data": [
    {
      "orderPublicId": "ORD-ABC123",
      "restaurantId": "...",
      "restaurantName": "Pizza Hut",
      "restaurantLogo": "https://...",
      "deliveredAt": "2026-02-20T..."
    },
    {
      "orderPublicId": "ORD-DEF456",
      "restaurantId": "...",
      "restaurantName": "KFC",
      "restaurantLogo": null,
      "deliveredAt": "2026-02-18T..."
    }
  ]
}
```

**Use this to:**
- Show a "Rate Order" button on delivered orders in the order history
- Show a badge/count of pending reviews

---

### 4. Touch Reminder — Prompt Customer to Review

Call this **once per app launch / home screen load**. The backend counts visits and decides when to prompt.

```
POST /api/v1/reviews/me/reminder/touch
Headers: Authorization: Bearer <customer_jwt>
```

**Response when NOT prompting (visits 1–5):**

```json
{
  "success": true,
  "data": {
    "shouldPrompt": false
  }
}
```

**Response when prompting (visit 6):**

```json
{
  "success": true,
  "data": {
    "shouldPrompt": true,
    "order": {
      "orderPublicId": "ORD-ABC123",
      "restaurantId": "...",
      "restaurantName": "Pizza Hut",
      "restaurantLogo": "https://...",
      "deliveredAt": "2026-02-20T..."
    }
  }
}
```

**When `shouldPrompt: true`** → show a bottom sheet / dialog:
> *"How was your order from Pizza Hut? Rate it!"*
> - Tap → open review form for that `orderPublicId`
> - Dismiss → nothing happens (won't prompt again for 24h)

---

### 5. Get Restaurant Reviews (Public)

Paginated list of reviews for a restaurant. **No auth required.**

```
GET /api/v1/reviews/restaurant/:restaurantId?limit=10&skip=0
```

| Param          | Type   | Default | Description                |
|----------------|--------|---------|----------------------------|
| `restaurantId` | string | —       | Restaurant ID (path param) |
| `limit`        | int    | 20      | Reviews per page           |
| `skip`         | int    | 0       | Offset for pagination      |

**Response:**

```json
{
  "success": true,
  "data": {
    "reviews": [
      {
        "id": "...",
        "rating": 5,
        "comment": "Best pizza in town!",
        "imageUrl": "https://...",
        "reviewerName": "John D.",
        "reviewerAvatar": "https://...",
        "createdAt": "2026-02-20T..."
      }
    ],
    "pagination": {
      "total": 42,
      "limit": 10,
      "skip": 0,
      "hasMore": true
    }
  }
}
```

> **Note:** No order info is exposed — only reviewer name, avatar, rating, comment, image, and date.

---

### 6. Get Restaurant Rating Stats (Public)

Get the aggregate rating for a restaurant. **No auth required.**

```
GET /api/v1/reviews/restaurant/:restaurantId/stats
```

**Response:**

```json
{
  "success": true,
  "data": {
    "rating": 4.3,
    "reviewCount": 42
  }
}
```

> **Note:** The restaurant list and detail endpoints (`GET /api/v1/restaurants` and `GET /api/v1/restaurants/details?slug=...`) already return live `rating` and `reviewCount` fields automatically. You do NOT need to call this endpoint separately unless you want to refresh stats independently.

---

## Review Reminder Flow

The backend handles **all anti-annoyance logic** server-side. The mobile app doesn't need to track anything locally.

### How it works internally

| Visit # | What Happens                                          |
|---------|-------------------------------------------------------|
| 1–5     | Counter increments, returns `shouldPrompt: false`     |
| 6       | Returns `shouldPrompt: true` + the most recent unreviewed order |
|         | Counter resets to 0                                   |
| 7–12    | Counter increments again...                           |
| 12      | Prompts again (if there's still a pending review)     |

### Anti-annoyance safeguards

| Rule                     | Description                                              |
|--------------------------|----------------------------------------------------------|
| **6-visit threshold**    | Won't prompt until 6th visit                             |
| **24h cooldown**         | Won't prompt again within 24 hours of last prompt        |
| **No repeat orders**     | Won't show the same order twice in a row                 |
| **No pending = no prompt** | If all orders are reviewed, returns `shouldPrompt: false` |

### Mobile implementation (pseudo-code)

```javascript
// Call ONCE on app launch or home screen mount
async function checkReviewReminder() {
  try {
    const response = await api.post('/reviews/me/reminder/touch');
    const { shouldPrompt, order } = response.data.data;

    if (shouldPrompt && order) {
      showReviewPrompt({
        orderPublicId: order.orderPublicId,
        restaurantName: order.restaurantName,
        restaurantLogo: order.restaurantLogo,
      });
    }
  } catch {
    // Silently ignore — not critical
  }
}
```

---

## Restaurant Detail Page — Live Ratings

The `rating` and `reviewCount` fields on restaurant endpoints are now **live** (computed from actual reviews):

```json
// GET /api/v1/restaurants/details?slug=pizza-hut
{
  "id": "...",
  "name": "Pizza Hut",
  "rating": 4.3,        // ← live average from reviews
  "reviewCount": 42,     // ← live count of published reviews
  ...
}
```

- If a restaurant has **0 reviews**, `rating` = `0` and `reviewCount` = `0`
- Rating is rounded to 1 decimal place (e.g., `4.3`, `3.7`)

---

## Testing with cURL / Postman

### Test 1: Submit a Review

```bash
# 1. Check which orders are pending review
curl https://your-api.com/api/v1/reviews/me/pending-orders \
  -H "Authorization: Bearer CUSTOMER_JWT"

# 2. Submit a review (use an orderPublicId from step 1)
curl -X POST https://your-api.com/api/v1/reviews \
  -H "Authorization: Bearer CUSTOMER_JWT" \
  -H "Content-Type: application/json" \
  -d '{
    "orderPublicId": "ORD-ABC123",
    "rating": 5,
    "comment": "Great food!"
  }'
```

### Test 2: Submit Review with Image

```bash
# 1. Upload image first
curl -X POST https://your-api.com/api/v1/reviews/upload-image \
  -H "Authorization: Bearer CUSTOMER_JWT" \
  -F "file=@/path/to/photo.jpg"
# Response: { "url": "https://storage.../review-images/abc.jpg" }

# 2. Submit review with the image URL
curl -X POST https://your-api.com/api/v1/reviews \
  -H "Authorization: Bearer CUSTOMER_JWT" \
  -H "Content-Type: application/json" \
  -d '{
    "orderPublicId": "ORD-ABC123",
    "rating": 4,
    "comment": "Tasty!",
    "imageUrl": "https://storage.../review-images/abc.jpg"
  }'
```

### Test 3: View Restaurant Reviews

```bash
# Public — no auth needed
curl "https://your-api.com/api/v1/reviews/restaurant/RESTAURANT_ID?limit=5&skip=0"
```

### Test 4: Test Reminder System

```bash
# Call 6 times — first 5 return shouldPrompt: false, 6th returns true
for i in {1..6}; do
  echo "Touch $i:"
  curl -s -X POST https://your-api.com/api/v1/reviews/me/reminder/touch \
    -H "Authorization: Bearer CUSTOMER_JWT" | jq .data
  echo ""
done
```

### Test 5: Duplicate Review (should fail)

```bash
# Try reviewing the same order again
curl -X POST https://your-api.com/api/v1/reviews \
  -H "Authorization: Bearer CUSTOMER_JWT" \
  -H "Content-Type: application/json" \
  -d '{
    "orderPublicId": "ORD-ABC123",
    "rating": 3
  }'
# Expected: 409 Conflict — "This order has already been reviewed"
```

---

## Error Handling

| Scenario                       | HTTP Status | Error Message                                           |
|--------------------------------|-------------|---------------------------------------------------------|
| Order not found                | 404         | `Order not found`                                       |
| Not your order                 | 400         | `You can only review your own orders`                   |
| Order not delivered            | 400         | `You can only review delivered orders`                  |
| Shop order (not restaurant)    | 400         | `Reviews are only available for restaurant orders`      |
| Already reviewed               | 409         | `This order has already been reviewed`                  |
| Rating out of range            | 400         | Validation error (must be 1–5 integer)                  |
| Missing rating                 | 400         | Validation error                                        |

---

## Summary

| Endpoint                                    | Auth     | Method | Purpose                              |
|---------------------------------------------|----------|--------|--------------------------------------|
| `POST /api/v1/reviews`                      | Customer | POST   | Submit a review                      |
| `POST /api/v1/reviews/upload-image`         | Customer | POST   | Upload review photo                  |
| `GET /api/v1/reviews/me/pending-orders`     | Customer | GET    | List unreviewed delivered orders     |
| `POST /api/v1/reviews/me/reminder/touch`    | Customer | POST   | Smart reminder (call on app launch)  |
| `GET /api/v1/reviews/restaurant/:id`        | Public   | GET    | List reviews for restaurant page     |
| `GET /api/v1/reviews/restaurant/:id/stats`  | Public   | GET    | Get rating + review count            |
