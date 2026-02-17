# 📱 Drovi Socket & API Documentation for Mobile Developers

> Complete reference for integrating Drovi's real-time features into mobile apps (React Native, Flutter, etc.)

---

## Table of Contents

1. [Server Info & Authentication](#1-server-info--authentication)
2. [Socket Architecture Overview](#2-socket-architecture-overview)
3. [How to Connect](#3-how-to-connect)
4. [Namespace 1: `/orders` — Full Reference](#4-namespace-1-orders--full-reference)
5. [Namespace 2: `/rider-location` — Full Reference](#5-namespace-2-rider-location--full-reference)
6. [REST API Endpoints (Order Lifecycle)](#6-rest-api-endpoints-order-lifecycle)
7. [Complete Order Flow (Step by Step)](#7-complete-order-flow-step-by-step)
8. [Role-Specific Integration Guides](#8-role-specific-integration-guides)
9. [Postman Testing Instructions](#9-postman-testing-instructions)
10. [Troubleshooting & FAQ](#10-troubleshooting--faq)

---

## 1. Server Info & Authentication

### Base URLs

| Environment | URL |
|---|---|
| Production | `https://drove-backend-f1d3892431b4.herokuapp.com` |
| REST API prefix | `/api/v1` |
| Socket `/orders` | `wss://drove-backend-f1d3892431b4.herokuapp.com/orders` |
| Socket `/rider-location` | `wss://drove-backend-f1d3892431b4.herokuapp.com/rider-location` |

### Login (Get JWT Token)

```
POST /api/v1/auth/login
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "password"
}
```

**Response:**
```json
{
  "user": {
    "id": "68dbbd4e7ba63e26ddb734ce",
    "email": "user@example.com",
    "role": "CUSTOMER",
    "isActive": true,
    "firstName": "Asim",
    "lastName": "Khan"
  },
  "tokens": {
    "accessToken": "eyJhbG...",
    "refreshToken": "eyJhbG..."
  }
}
```

**Important:** The `user.id` from login response = `userId`. This is what you use for ALL socket registrations.

### Test Accounts

| Role | Email | Password | userId |
|---|---|---|---|
| Customer | idmery77@gmail.com | AsimRauf678# | `68dbbd4e7ba63e26ddb734ce` |
| Rider | asimraufbuzz@gmail.com | AsimRauf678# | `690ba26be8e0b05bd9f1d8a0` |
| Restaurant | mainlynotdoingso@gmail.com | AsimRauf678# | `691e26ce1d88bac5205b0bda` |

Restaurant's `vendorId` (used for socket registration): `691e26ce1d88bac5205b0bdb`

> ⚠️ **`userId` vs `vendorId`**: The `userId` is the user's auth ID from login. The `vendorId` is the restaurant/shop's own ID. For vendor socket registration, you need the **vendorId**, not the userId.

---

## 2. Socket Architecture Overview

Drovi has **2 separate WebSocket namespaces**. Think of them as 2 independent socket connections.

### `/orders` — Order Lifecycle Events
- Used by: **Everyone** (Customer, Rider, Restaurant, Admin)
- Purpose: Order notifications, status updates, rider assignment, payment events

### `/rider-location` — GPS Tracking
- Used by: **Rider app** (sends GPS), **Admin dashboard** (monitors riders)
- Purpose: Real-time rider location, status (online/offline/busy), nearby rider search

### Who Needs What

| Role | `/orders` | `/rider-location` | Total Connections |
|---|---|---|---|
| Customer | ✅ (register_customer) | ❌ | 1 |
| Rider | ✅ (register_rider) | ✅ (register_rider + GPS) | 2 |
| Restaurant/Shop | ✅ (register_vendor) | ❌ | 1 |
| Admin | ✅ (register_admin) | ✅ (optional, for live map) | 1-2 |

---

## 3. How to Connect

### Socket.IO Client Setup

Use `socket.io-client` v4. Works with React Native, Flutter (via dart socket_io_client), etc.

```javascript
import { io } from 'socket.io-client';

const socket = io('wss://drove-backend-f1d3892431b4.herokuapp.com/orders', {
  transports: ['websocket', 'polling'],  // websocket preferred, polling as fallback
  auth: {
    token: 'YOUR_JWT_ACCESS_TOKEN',      // optional today, may be enforced later
  },
  reconnection: true,                     // auto-reconnect on disconnect
  reconnectionDelay: 1000,                // wait 1s before first retry
  reconnectionAttempts: Infinity,         // never stop retrying
  timeout: 20000,                         // connection timeout: 20s
});
```

### Connection Lifecycle

```javascript
socket.on('connect', () => {
  console.log('Connected! Socket ID:', socket.id);
  // IMPORTANT: Register your role HERE (after every connect/reconnect)
});

socket.on('disconnect', (reason) => {
  console.log('Disconnected:', reason);
  // If server kicked us, manually reconnect
  if (reason === 'io server disconnect') {
    socket.connect();
  }
});

socket.on('connect_error', (error) => {
  console.error('Connection error:', error.message);
});
```

### Critical Rule: Re-register After Every Reconnect

Socket.IO may reconnect automatically (network drop, server restart). After EVERY reconnect, you MUST re-register your role. The server forgets you on disconnect.

```javascript
socket.on('connect', () => {
  // Always re-register here
  socket.emit('register_customer', { customerId: userId }, (response) => {
    console.log('Registered:', response);
  });
});
```

---

## 4. Namespace 1: `/orders` — Full Reference

### 4.1 Events You EMIT (client → server)

---

#### `ping`
Health check. Confirms the connection is alive.

```javascript
socket.emit('ping', {}, (response) => {
  // response: { success: true }
});
// Also triggers a 'pong' event back
```

| Field | Type | Required | Description |
|---|---|---|---|
| (none) | — | — | Send empty object `{}` |

**Response (ack):** `{ success: true }`
**Also emits back:** `pong` event (no payload)

---

#### `register_admin`
Register as an admin to receive payment alerts, escalations, and order notifications for your assigned city.

```javascript
socket.emit('register_admin', { userId: 'ADMIN_USER_ID' }, (response) => {
  // response: { success: true, adminId: '...' }
});
```

| Field | Type | Required | Description |
|---|---|---|---|
| `userId` | string | ✅ | Admin's userId from login |

**Response (ack):** `{ success: true, adminId: "..." }` or `{ success: false, error: "..." }`

**Server behavior:** Looks up admin's `assignedCity`. Admin will ONLY receive events for orders in that city.

---

#### `register_customer`
Register as a customer to receive order updates and rider location.

```javascript
socket.emit('register_customer', { customerId: 'USER_ID' }, (response) => {
  // response: { success: true, customerId: '...' }
});
```

| Field | Type | Required | Description |
|---|---|---|---|
| `customerId` | string | ✅ | Customer's userId from login |

**Response (ack):** `{ success: true, customerId: "..." }`

---

#### `register_rider`
Register as a rider to receive new order notifications.

```javascript
socket.emit('register_rider', { riderId: 'USER_ID' }, (response) => {
  // response: { success: true, riderId: '...' }
});
```

| Field | Type | Required | Description |
|---|---|---|---|
| `riderId` | string | ✅ | Rider's userId from login |

**Response (ack):** `{ success: true, riderId: "..." }`

> ⚠️ **Rider must ALSO register on `/rider-location` namespace separately** for GPS tracking.

---

#### `register_vendor`
Register as a restaurant or shop to receive order notifications.

```javascript
socket.emit('register_vendor', {
  vendorType: 'restaurant',
  vendorId: '691e26ce1d88bac5205b0bdb'
}, (response) => {
  // response: { success: true, room: 'vendor:restaurant:691e26ce1d88bac5205b0bdb' }
});
```

| Field | Type | Required | Description |
|---|---|---|---|
| `vendorType` | `"restaurant"` or `"shop"` | ✅ | Type of vendor |
| `vendorId` | string | ✅ | The restaurant's or shop's ID (NOT the userId!) |

**Response (ack):** `{ success: true, room: "vendor:restaurant:..." }`

**Server behavior:** Joins the socket to a room `vendor:{type}:{id}`. All order events for this vendor are broadcast to this room, so multiple devices/tabs can receive simultaneously.

> **How to get vendorId:** After login with restaurant email, call `GET /api/v1/restaurants/my-restaurant` or get it from the restaurant profile.

---

#### `subscribe_to_order`
Customer subscribes to updates for a specific order.

```javascript
socket.emit('subscribe_to_order', {
  orderId: 'ORDER_ID',
  customerId: 'USER_ID'
}, (response) => {
  // response: { success: true, orderId: '...' }
});
```

| Field | Type | Required | Description |
|---|---|---|---|
| `orderId` | string | ✅ | The order's ID |
| `customerId` | string | ✅ | Customer's userId |

**Response (ack):** `{ success: true, orderId: "..." }`

**When to use:** After placing an order, subscribe to get updates for that specific order. This is an ADDITIONAL subscription on top of `register_customer` — if you register as customer, you already receive events sent directly to your customerId. This subscription is for orders that use broadcast-based delivery.

---

#### `unsubscribe_from_order`
Stop receiving updates for a specific order.

```javascript
socket.emit('unsubscribe_from_order', { orderId: 'ORDER_ID' }, (response) => {
  // response: { success: true, orderId: '...' }
});
```

| Field | Type | Required | Description |
|---|---|---|---|
| `orderId` | string | ✅ | The order's ID |

---

### 4.2 Events You LISTEN TO (server → client)

---

#### `pong`
Response to `ping`. Confirms connection is alive.

**Who receives:** The socket that sent `ping`
**Payload:** none

```javascript
socket.on('pong', () => {
  console.log('Connection is alive');
});
```

---

#### `order_ready`
A new confirmed/paid order is ready for the vendor to see.

**Who receives:** Restaurant/Shop (in their vendor room)
**When:** 
- CASH order confirmed → sent immediately
- EVC_PLUS order → sent after admin verifies payment
- CARD order → sent after Square webhook confirms payment
- Also sent when vendor clicks "Start Preparing" (state sync)
- Also sent when order is delivered (final state sync)

**Payload:** Full enriched order object

```javascript
socket.on('order_ready', (order) => {
  console.log('New order!', order.publicId, order.status);
  // order = {
  //   id: "699419ad8dff2554a3f47f2f",
  //   publicId: "#DRV-RI092",
  //   status: "CONFIRMED",
  //   vendorType: "restaurant",
  //   paymentMethod: "CASH",
  //   subtotal: 393,
  //   deliveryFee: 0,
  //   total: 393,
  //   items: [...],
  //   customer: { id, name, phone, email },
  //   vendor: { id, name, phone, address },
  //   delivery: { latitude, longitude, address, distance },
  //   riderInfo: null | { id, name, phone, location },
  //   ...
  // }
});
```

---

#### `new_payment`
A customer submitted a payment that needs admin verification (EVC_PLUS only).

**Who receives:** Admin (same city as order)
**When:** Customer confirms order with EVC_PLUS payment method

**Payload:** Full order object (same structure as `order_ready`)

```javascript
socket.on('new_payment', (order) => {
  console.log('Payment to verify:', order.publicId, order.total);
  // Show in admin "Pending Payments" list
});
```

---

#### `payment_verified`
Admin has verified a payment.

**Who receives:** Admin (same city)
**When:** Admin calls `POST /orders/admin/verify-payment/:orderId`

```javascript
socket.on('payment_verified', (data) => {
  // data = { orderId: "...", adminId: "..." }
  // Remove from pending payments list
});
```

---

#### `payment_rejected`
Admin has rejected a payment.

**Who receives:** Admin (same city)
**When:** Admin calls `POST /orders/admin/reject-payment/:orderId`

```javascript
socket.on('payment_rejected', (data) => {
  // data = { orderId: "...", adminId: "...", reason: "Invalid proof" }
});
```

---

#### `order_status_updated`
Order status has changed. This is the MAIN event customers listen to.

**Who receives:** Customer (sent directly to registered customerId OR to subscribed customers)
**When:** ANY status change — CONFIRMED, PREPARING, READY_FOR_PICKUP, ACCEPTED_BY_RIDER, PICKED_UP, DELIVERED, CANCELLED

**Payload:**
```javascript
socket.on('order_status_updated', (data) => {
  // data = {
  //   orderId: "#DRV-RI092",     // publicId
  //   status: "PREPARING",
  //   timestamp: 1771314939020,
  //   // ...plus full enriched order data:
  //   id: "699419ad...",
  //   items: [...],
  //   customer: {...},
  //   vendor: {...},
  //   delivery: {...},
  //   riderInfo: { id, name, phone, location } | null,
  //   total: 393,
  //   paymentMethod: "CASH",
  //   ...
  // }
});
```

**Important:** The `orderId` field in this event is the `publicId` (e.g., `#DRV-RI092`), NOT the database ID. The database ID is in the `id` field.

---

#### `rider_assigned`
A rider has been assigned to the customer's order.

**Who receives:** Customer
**When:** Rider accepts the order

```javascript
socket.on('rider_assigned', (data) => {
  // data = { orderId: "699419ad...", riderId: "690ba26b..." }
  // Show "Rider is on the way" UI
});
```

---

#### `rider_location_update`
Rider's GPS position update for a specific order.

**Who receives:** Customer (on `/orders` namespace), Restaurant/Shop (in vendor room)
**When:** Rider sends GPS update AND rider has an active order (PREPARING, READY_FOR_PICKUP, ACCEPTED_BY_RIDER, PICKED_UP, OUT_FOR_DELIVERY)

```javascript
socket.on('rider_location_update', (data) => {
  // data = {
  //   orderId: "699419ad8dff2554a3f47f2f",
  //   location: {
  //     latitude: 28.3100,
  //     longitude: 70.1100,
  //     timestamp: 1771314939678
  //   }
  // }
  // Update rider marker on map
  updateMapMarker(data.location.latitude, data.location.longitude);
});
```

**How it works internally:**
1. Rider sends `update_location` on `/rider-location` namespace
2. Backend finds rider's active orders in database
3. For each active order, sends `rider_location_update` to the customer on `/orders` namespace
4. Also sends to the vendor's room on `/orders` namespace

---

#### `new_order_nearby`
A new order is available for the rider to accept.

**Who receives:** Rider (on `/orders` namespace)
**When:** Restaurant clicks "Start Preparing" → system finds top 4 nearest online riders → sends to each

```javascript
socket.on('new_order_nearby', (data) => {
  // data = {
  //   orderId: "699419ad8dff2554a3f47f2f",
  //   publicId: "#DRV-RI092",
  //   vendorType: "restaurant",
  //   vendorName: "Cafe Sajawal",
  //   vendorLogo: "https://storage.googleapis.com/...",
  //   customerName: "Asim Khan",
  //   pickupAddress: {
  //     street: "Shadman Town Street No 4",
  //     city: "Sadiqabad",
  //     latitude: 28.302646,
  //     longitude: 70.126787,
  //     formattedAddress: "Shadman Town Street No 4, Sadiqabad, Punjab"
  //   },
  //   deliveryAddress: {
  //     latitude: 28.3136584,
  //     longitude: 70.0992775,
  //     formattedAddress: "Sadiqabad Bypass, Habib Colony, Sadiqabad, Pakistan"
  //   },
  //   items: [
  //     { name: "Proper Deal Package", quantity: 1 }
  //   ],
  //   total: 393,
  //   deliveryFee: 0,             // Always 0 (hidden from rider)
  //   estimatedEarnings: 1.05,    // Rider's commission amount
  //   commissionPercent: 70,      // Rider's commission %
  //   commissionAmount: 1.05,     // Actual $ amount rider earns
  //   riderTip: 0,
  //   paymentMethod: "CASH",
  //   distance: "1.84",           // km from rider to delivery point
  //   createdAt: "2026-02-17T07:33:01.201Z"
  // }
  
  // Show order card to rider with Accept/Reject buttons
  showOrderNotification(data);
});
```

**Retry behavior:** If no rider accepts, the server retries every 2 minutes, up to 5 times (10 minutes total). Each retry re-searches for nearby riders and sends `new_order_nearby` again. Riders who already rejected are excluded.

---

#### `order_no_longer_available`
Another rider accepted this order. Remove it from your available orders list.

**Who receives:** Rider (all other riders who were notified about this order)
**When:** One rider accepts → all others get this

```javascript
socket.on('order_no_longer_available', (data) => {
  // data = {
  //   orderId: "699419ad8dff2554a3f47f2f",
  //   reason: "accepted_by_another_rider",
  //   message: "This order was picked up by another rider"
  // }
  
  // Remove this order from the available orders list
  removeOrderFromList(data.orderId);
});
```

---

#### `order_accepted`
Your acceptance of an order has been confirmed. The order is now yours.

**Who receives:** The specific rider who accepted
**When:** Server confirms the rider's accept request succeeded

```javascript
socket.on('order_accepted', (data) => {
  // data = {
  //   orderId: "699419ad8dff2554a3f47f2f",
  //   order: { ...full enriched order with vendor/customer/delivery info... },
  //   timestamp: 1771314939020
  // }
  
  // Navigate to active delivery screen
  navigateToActiveDelivery(data.order);
});
```

**vs `order_no_longer_available`:** These 2 events are for DIFFERENT riders.
- Rider A presses Accept → Rider A gets `order_accepted` ✅
- Riders B, C, D (who were also notified) → get `order_no_longer_available` ❌

---

#### `order_status_update`
Status change for the rider's currently active order.

**Who receives:** Rider (for their assigned order)
**When:** Order status changes while rider is assigned (e.g., READY_FOR_PICKUP, PREPARING)

```javascript
socket.on('order_status_update', (data) => {
  // data = {
  //   orderId: "699419ad8dff2554a3f47f2f",
  //   status: "READY_FOR_PICKUP",
  //   timestamp: 1771314939020,
  //   ...optional enriched order data
  // }
  
  // Update order status in rider's active delivery screen
  updateOrderStatus(data.status);
});
```

**⚠️ Note:** This is `order_status_update` (for riders), NOT `order_status_updated` (for customers). They are different events with slightly different payloads.

---

#### `rider_accepted`
A rider has accepted an order from this vendor.

**Who receives:** Restaurant/Shop (in their vendor room)
**When:** Rider accepts an order assigned to this restaurant/shop

```javascript
socket.on('rider_accepted', (data) => {
  // data = full enriched order including:
  // {
  //   id: "...",
  //   publicId: "#DRV-RI092",
  //   status: "PREPARING",
  //   riderInfo: {
  //     id: "690ba26be...",
  //     name: "Asim Rauf",
  //     phone: "+16342673463",
  //     email: "asimraufbuzz@gmail.com",
  //     location: { latitude: 28.302, longitude: 70.126 },
  //     commissionPercent: 70,
  //     commissionAmount: 1.05
  //   },
  //   ...
  // }
  
  // Show rider info on the order card
  showRiderOnOrder(data.publicId, data.riderInfo);
});
```

---

#### `order_needs_attention`
No riders are available for an order. Needs admin intervention.

**Who receives:** Admin (same city as order)
**When:** No online riders found, or all riders have insufficient wallet balance for a CASH order

```javascript
socket.on('order_needs_attention', (data) => {
  // data = {
  //   orderId: "699419ad8dff2554a3f47f2f",
  //   reason: "No online riders available",
  //   timestamp: 1771314939020
  // }
  
  // Show alert in admin dashboard
  showAdminAlert(data);
});
```

---

#### `order_escalation`
**HIGH PRIORITY.** No rider accepted after 10 minutes. Includes full details for admin to manually assign or contact riders.

**Who receives:** Admin (same city as order)
**When:** 5 retry cycles (2 min each) exhausted with no acceptance

```javascript
socket.on('order_escalation', (data) => {
  // data = {
  //   orderId: "699419ad8dff2554a3f47f2f",
  //   publicId: "#DRV-RI092",
  //   orderTotal: 393,
  //   restaurantName: "Cafe Sajawal",
  //   restaurantPhone: "+12312312312",
  //   customerName: "Asim Khan",
  //   customerPhone: "+923013978580",
  //   deliveryAddress: "Sadiqabad Bypass, Habib Colony, Sadiqabad, Pakistan",
  //   nearbyRidersCount: 1,
  //   nearbyRiderPhones: ["+16342673463"],
  //   reason: "No rider accepted order after 10 minutes",
  //   createdAt: "2026-02-17T07:33:01.201Z",
  //   cityOfOrder: "Sadiqabad",
  //   priority: "HIGH",
  //   timestamp: 1771314939020
  // }
  
  // Show urgent alert with phone numbers to call
  showUrgentEscalation(data);
});
```

---

#### `new_deposit_request`
A rider has requested a wallet deposit.

**Who receives:** Admin (same city as rider, or all admins if no city set)
**When:** Rider submits deposit request via REST API

```javascript
socket.on('new_deposit_request', (depositData) => {
  // depositData = deposit record object
  // Show in admin "Pending Deposits" section
});
```

---

#### `new_withdrawal_request`
A rider has requested a wallet withdrawal.

**Who receives:** Admin (same city as rider, or all admins if no city set)
**When:** Rider submits withdrawal request via REST API

```javascript
socket.on('new_withdrawal_request', (withdrawalData) => {
  // withdrawalData = withdrawal record object
  // Show in admin "Pending Withdrawals" section
});
```

---

## 5. Namespace 2: `/rider-location` — Full Reference

### 5.1 Events You EMIT (client → server)

---

#### `register_rider`
Register rider for GPS tracking. Marks rider as online.

```javascript
locationSocket.emit('register_rider', { riderId: 'USER_ID' }, (response) => {
  // response: { success: true, riderId: '...', status: 'online' }
});
```

| Field | Type | Required | Description |
|---|---|---|---|
| `riderId` | string | ✅ | Rider's userId from login |

**Response (ack):** `{ success: true, riderId: "...", status: "online" }`

**Server behavior:**
- Stores socket↔riderId mapping
- Marks rider as ONLINE in database
- Cancels any pending disconnect timer (if rider reconnected within 5s grace period)
- Emits `rider_status_updated` back to the rider

---

#### `update_location`
Send rider's current GPS coordinates. **Call this frequently** (every time GPS changes).

```javascript
locationSocket.emit('update_location', {
  riderId: 'USER_ID',
  latitude: 28.302646,
  longitude: 70.126787,
  heading: 180,       // compass direction in degrees (0=North, 90=East)
  speed: 25.5,        // meters per second
  accuracy: 10,       // GPS accuracy in meters
}, (response) => {
  // response: { success: true, timestamp: 1771314939020 }
});
```

| Field | Type | Required | Description |
|---|---|---|---|
| `riderId` | string | ✅ | Rider's userId |
| `latitude` | number | ✅ | GPS latitude |
| `longitude` | number | ✅ | GPS longitude |
| `heading` | number | ❌ | Compass heading (degrees, 0-360) |
| `speed` | number | ❌ | Speed in m/s |
| `accuracy` | number | ❌ | GPS accuracy in meters |

**Response (ack):** `{ success: true, timestamp: 1771314939020 }`

**Server behavior (important!):**
1. Saves location to Redis (geospatial index)
2. Broadcasts `rider_location_updated` to ALL clients on `/rider-location` (for admin map)
3. Sends `rider_location_update` to admin subscribers watching this specific rider
4. **Finds all active orders for this rider** in the database
5. For each active order: sends `rider_location_update` to the **customer** and **vendor** on `/orders` namespace

So when a rider sends GPS, the customer automatically sees the rider moving on the map. No extra work needed from the customer side.

**Recommended frequency:** Use `navigator.geolocation.watchPosition()` or platform equivalent. This auto-fires when the device GPS changes. Typically sends every 1-10 seconds while moving.

---

#### `update_status`
Change rider's availability status.

```javascript
locationSocket.emit('update_status', {
  riderId: 'USER_ID',
  status: 'online'    // 'online' | 'offline' | 'busy'
}, (response) => {
  // response: { success: true }
});
```

| Field | Type | Required | Description |
|---|---|---|---|
| `riderId` | string | ✅ | Rider's userId |
| `status` | string | ✅ | `"online"`, `"offline"`, or `"busy"` |

**Status meanings:**
- `online` — Available to receive new order notifications
- `offline` — Not working, won't receive order notifications
- `busy` — Currently on a delivery (set automatically when rider accepts an order)

**Response (ack):** `{ success: true }`

**Server behavior:** Broadcasts `rider_status_updated` to ALL clients on `/rider-location`.

---

#### `admin_subscribe_rider`
Admin subscribes to detailed location updates for a specific rider.

```javascript
locationSocket.emit('admin_subscribe_rider', { riderId: 'RIDER_USER_ID' }, (response) => {
  // response: { success: true }
});
```

| Field | Type | Required | Description |
|---|---|---|---|
| `riderId` | string | ✅ | The rider's userId to track |

**After subscribing:** Admin receives `rider_location_update` events whenever that specific rider moves.

---

#### `find_nearby_riders`
Search for nearby available riders by coordinates.

```javascript
locationSocket.emit('find_nearby_riders', {
  latitude: 28.302646,
  longitude: 70.126787,
  radiusKm: 50,
  limit: 10
}, (response) => {
  // response: {
  //   success: true,
  //   riders: [
  //     {
  //       riderId: "690ba26be8e0b05bd9f1d8a0",
  //       firstName: "Asim",
  //       lastName: "Rauf",
  //       avatar: "https://...",
  //       distance: 1.836,
  //       location: { latitude: 28.31, longitude: 70.11 },
  //       status: "online",
  //       vehicleType: "MOTORCYCLE",
  //       isAvailable: true
  //     }
  //   ],
  //   count: 1
  // }
});
```

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `latitude` | number | ✅ | — | Center latitude |
| `longitude` | number | ✅ | — | Center longitude |
| `radiusKm` | number | ❌ | 15 | Search radius in km |
| `limit` | number | ❌ | 20 | Max riders to return |

---

#### `get_rider_location`
Get a specific rider's current location.

```javascript
locationSocket.emit('get_rider_location', { riderId: 'RIDER_USER_ID' }, (response) => {
  // response: {
  //   success: true,
  //   location: {
  //     riderId: "690ba26be...",
  //     latitude: 28.31,
  //     longitude: 70.11,
  //     timestamp: 1771314939678,
  //     status: "online",
  //     heading: 180,
  //     speed: 25,
  //     accuracy: 5
  //   }
  // }
});
```

| Field | Type | Required | Description |
|---|---|---|---|
| `riderId` | string | ✅ | The rider's userId |

---

### 5.2 Events You LISTEN TO (server → client)

---

#### `rider_status_updated`
A rider's status changed (online/offline/busy).

**Who receives:** ALL connected clients on `/rider-location`
**When:** Any rider calls `update_status` or registers/disconnects

```javascript
locationSocket.on('rider_status_updated', (data) => {
  // data = { riderId: "690ba26be...", status: "online" }
});
```

---

#### `rider_location_updated`
A rider's GPS position changed. Broadcast to ALL clients on this namespace.

**Who receives:** ALL connected clients on `/rider-location` (used for admin live map)
**When:** Any rider calls `update_location`

```javascript
locationSocket.on('rider_location_updated', (data) => {
  // data = {
  //   riderId: "690ba26be...",
  //   latitude: 28.302646,
  //   longitude: 70.126787,
  //   timestamp: 1771314939020,
  //   status: "online",
  //   heading: 180,
  //   speed: 25,
  //   accuracy: 5
  // }
  // Update rider pin on admin map
});
```

---

#### `rider_location_update`
Detailed location update for a specific rider. Only sent to admin subscribers.

**Who receives:** Admins who called `admin_subscribe_rider` for this rider
**When:** The subscribed rider sends `update_location`

```javascript
locationSocket.on('rider_location_update', (data) => {
  // data = full RiderLocation object (same as rider_location_updated)
});
```

---

#### `new_order_nearby`
New order notification (also sent on this namespace via `notifyNearbyRidersOfOrder`).

**Who receives:** Nearby riders found by geospatial search
**When:** New order created near this rider

```javascript
locationSocket.on('new_order_nearby', (data) => {
  // data = { orderId, distance, orderDetails }
});
```

> ⚠️ In practice, the main `new_order_nearby` event comes on the `/orders` namespace with richer data. This one on `/rider-location` is a secondary notification path.

---

## 6. REST API Endpoints (Order Lifecycle)

These are NOT socket events — they are HTTP REST calls. Socket events are triggered as SIDE EFFECTS of these API calls.

### Base URL: `https://drove-backend-f1d3892431b4.herokuapp.com/api/v1`

All authenticated endpoints require:
```
Authorization: Bearer <accessToken>
```

---

### Customer Endpoints

| Method | Endpoint | Auth | Description | Socket Side Effects |
|---|---|---|---|---|
| POST | `/orders/draft` | Customer | Create draft order | None |
| GET | `/orders/draft/:draftId` | Customer | Get draft details | None |
| POST | `/orders/upload-payment-proof` | Customer | Upload EVC payment screenshot | None |
| POST | `/orders/confirm` | Customer | Confirm & pay for order | `new_payment` → Admin, `order_ready` → Vendor, `order_status_updated` → Customer |
| GET | `/orders/my-orders` | Customer | Get all customer's orders | None |

#### Confirm Order Body:
```json
{
  "draftId": "draft-public-id",
  "paymentMethod": "CASH",
  "transactionId": null,
  "paymentProofUrl": null,
  "riderTip": 0,
  "squarePaymentToken": null
}
```

Payment methods: `"CASH"`, `"EVC_PLUS"`, `"CARD"`

---

### Vendor Endpoints

| Method | Endpoint | Auth | Description | Socket Side Effects |
|---|---|---|---|---|
| POST | `/orders/vendor/:orderId/start-preparing` | Restaurant/Shop | Mark order as PREPARING | `order_status_updated` → Customer, `new_order_nearby` → Top 4 riders |
| POST | `/orders/vendor/:orderId/mark-prepared` | Restaurant/Shop | Mark as READY_FOR_PICKUP | `order_status_updated` → Customer, `order_status_update` → Assigned rider |
| POST | `/orders/vendor/:orderId/mark-picked-up` | Restaurant/Shop | Customer picked up (pickup orders) | `order_status_updated` → Customer |
| GET | `/orders/vendor/my-orders` | Restaurant/Shop | Get vendor's orders | None |

---

### Rider Endpoints

| Method | Endpoint | Auth | Description | Socket Side Effects |
|---|---|---|---|---|
| GET | `/orders/rider/available` | Rider | Get available orders | None |
| POST | `/orders/rider/:orderId/accept` | Rider | Accept an order | `order_accepted` → This rider, `order_no_longer_available` → Other riders, `rider_accepted` → Vendor, `order_status_updated` + `rider_assigned` → Customer |
| POST | `/orders/rider/:orderId/reject` | Rider | Reject an order | None (rider excluded from future retries) |
| POST | `/orders/rider/:orderId/delivered` | Rider | Mark as delivered | `order_status_updated` → Customer, `order_ready` → Vendor (final sync) |
| GET | `/orders/rider/active-deliveries` | Rider | Get active/completed deliveries | None |

#### Accept Order Response:
```json
{
  "success": true,
  "order": {
    "id": "...",
    "publicId": "#DRV-RI092",
    "status": "PREPARING",
    "riderInfo": {
      "id": "...",
      "name": "Asim Rauf",
      "phone": "+16342673463"
    }
  }
}
```

#### Accept Order Failure Cases:
| Response | Reason |
|---|---|
| `"Order already picked up by another rider"` | Race condition — someone else accepted first |
| `"Order cannot be accepted in status: DELIVERED"` | Order already progressed past acceptance |
| `"You can only accept orders in Sadiqabad"` | Rider's city ≠ order's city |
| `"Insufficient wallet balance..."` | CASH order but rider's wallet is too low |
| `"Rider profile not found"` | userId doesn't have a rider profile |

---

### Admin Endpoints

| Method | Endpoint | Auth | Description | Socket Side Effects |
|---|---|---|---|---|
| GET | `/orders/admin/pending-payments` | Admin | List pending EVC payments | None |
| POST | `/orders/admin/verify-payment/:orderId` | Admin | Verify EVC payment | `payment_verified` → Admin, `order_status_updated` → Customer, `order_ready` → Vendor |
| POST | `/orders/admin/reject-payment/:orderId` | Admin | Reject EVC payment | `payment_rejected` → Admin, `order_status_updated` → Customer |
| GET | `/orders/admin/all-orders` | Admin | List all orders (paginated) | None |

---

## 7. Complete Order Flow (Step by Step)

### Order Status Progression:

```
PENDING → CONFIRMED → PREPARING → READY_FOR_PICKUP → PICKED_UP → DELIVERED
                                                         ↑
                                               (or ACCEPTED_BY_RIDER here)
```

### Flow by Payment Method:

#### CASH Orders (simplest):
```
1. Customer: POST /orders/confirm { paymentMethod: "CASH" }
2. Server:   order_ready → Restaurant
3. Server:   order_status_updated (CONFIRMED) → Customer
4. Restaurant: POST /orders/vendor/:id/start-preparing
5. Server:   order_status_updated (PREPARING) → Customer
6. Server:   new_order_nearby → Top 4 nearest riders
7. Rider: POST /orders/rider/:id/accept
8. Server:   order_accepted → This rider
9. Server:   order_no_longer_available → Other riders
10. Server:  rider_accepted → Restaurant
11. Server:  order_status_updated → Customer
12. Server:  rider_assigned → Customer
13. Restaurant: POST /orders/vendor/:id/mark-prepared
14. Server:  order_status_updated (READY_FOR_PICKUP) → Customer + Rider
15. Rider picks up, delivers...
16. Rider: POST /orders/rider/:id/delivered
17. Server:  order_status_updated (DELIVERED) → Customer + Restaurant
```

#### EVC_PLUS Orders (admin verification):
```
1. Customer: POST /orders/confirm { paymentMethod: "EVC_PLUS", transactionId, paymentProofUrl }
2. Server:   new_payment → Admin (same city)
3. Server:   order_status_updated (PENDING) → Customer
4. Admin: POST /orders/admin/verify-payment/:id
5. Server:   payment_verified → Admin
6. Server:   order_ready → Restaurant
7. Server:   order_status_updated (CONFIRMED) → Customer
   ... (then same as CASH from step 4 onwards)
```

#### CARD Orders (Square webhook):
```
1. Customer: POST /orders/confirm { paymentMethod: "CARD", squarePaymentToken }
2. Server:   Sends payment to Square, order stays PENDING
3. Square webhook fires → Server confirms payment
4. Server:   order_ready → Restaurant
5. Server:   order_status_updated (CONFIRMED) → Customer
   ... (then same as CASH from step 4 onwards)
```

### Rider Notification Retry Timeline:

```
T+0min   Restaurant clicks "Start Preparing"
         → new_order_nearby sent to top 4 nearest riders
         
T+2min   Retry 1: Re-search, notify again (excluding riders who rejected)
T+4min   Retry 2: Re-search, notify again
T+6min   Retry 3: Re-search, notify again
T+8min   Retry 4: Re-search, notify again
T+10min  Retry 5: Final attempt

T+10min  ESCALATION: order_escalation → Admin with full details + phone numbers
         Order marked as escalatedToAdmin = true
```

### Disconnect Grace Period:

When a rider's socket disconnects (network drop, app backgrounded):
1. Server waits **5 seconds** before marking rider offline
2. If rider reconnects within 5s → timer is cancelled, rider stays online
3. If rider doesn't reconnect → marked offline, stops receiving order notifications

---

## 8. Role-Specific Integration Guides

### 8.1 Rider App Integration

```javascript
// ========================================
// RIDER APP — COMPLETE SOCKET INTEGRATION
// ========================================

import { io } from 'socket.io-client';

const SERVER = 'https://drove-backend-f1d3892431b4.herokuapp.com';
let ordersSocket, locationSocket;

// === CONNECT (call once after login) ===
function connectRider(userId, token) {
  // Connection 1: Order notifications
  ordersSocket = io(`${SERVER}/orders`, {
    transports: ['websocket', 'polling'],
    auth: { token },
    reconnection: true,
    reconnectionDelay: 1000,
  });

  // Connection 2: GPS tracking
  locationSocket = io(`${SERVER}/rider-location`, {
    transports: ['websocket', 'polling'],
    auth: { token },
    reconnection: true,
    reconnectionDelay: 1000,
  });

  // --- Register on BOTH after every connect ---
  ordersSocket.on('connect', () => {
    ordersSocket.emit('register_rider', { riderId: userId }, (res) => {
      console.log('Registered on /orders:', res);
    });
  });

  locationSocket.on('connect', () => {
    locationSocket.emit('register_rider', { riderId: userId }, (res) => {
      console.log('Registered on /rider-location:', res);
      // Go online
      locationSocket.emit('update_status', { riderId: userId, status: 'online' });
    });
  });

  // --- LISTEN: New order available ---
  ordersSocket.on('new_order_nearby', (data) => {
    // data.orderId, data.vendorName, data.distance, data.estimatedEarnings,
    // data.pickupAddress, data.deliveryAddress, data.items, data.paymentMethod
    showNewOrderCard(data);
  });

  // --- LISTEN: Your acceptance confirmed ---
  ordersSocket.on('order_accepted', (data) => {
    // data.orderId, data.order (full order)
    navigateToActiveDelivery(data.order);
  });

  // --- LISTEN: Someone else took it ---
  ordersSocket.on('order_no_longer_available', (data) => {
    // data.orderId, data.reason, data.message
    removeOrderCard(data.orderId);
    showToast('Order taken by another rider');
  });

  // --- LISTEN: Active order status change ---
  ordersSocket.on('order_status_update', (data) => {
    // data.orderId, data.status, data.timestamp
    updateActiveOrderStatus(data.status);
  });

  // --- LISTEN: Status confirmation ---
  locationSocket.on('rider_status_updated', (data) => {
    // data.riderId, data.status
    updateStatusUI(data.status);
  });
}

// === SEND GPS (call from geolocation watcher) ===
function sendLocation(userId, coords) {
  locationSocket?.emit('update_location', {
    riderId: userId,
    latitude: coords.latitude,
    longitude: coords.longitude,
    heading: coords.heading || 0,
    speed: coords.speed || 0,
    accuracy: coords.accuracy || 0,
  });
}

// === ACCEPT ORDER (REST API, not socket) ===
async function acceptOrder(orderId, token) {
  const res = await fetch(`${SERVER}/api/v1/orders/rider/${orderId}/accept`, {
    method: 'POST',
    headers: { Authorization: `Bearer ${token}` },
  });
  return res.json();
  // If success → order_accepted event will arrive on socket
  // If fail → response.message tells you why
}

// === REJECT ORDER (REST API) ===
async function rejectOrder(orderId, token, reason) {
  const res = await fetch(`${SERVER}/api/v1/orders/rider/${orderId}/reject`, {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${token}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({ reason }),
  });
  return res.json();
}

// === MARK DELIVERED (REST API) ===
async function markDelivered(orderId, token) {
  const res = await fetch(`${SERVER}/api/v1/orders/rider/${orderId}/delivered`, {
    method: 'POST',
    headers: { Authorization: `Bearer ${token}` },
  });
  return res.json();
  // order_status_updated → Customer & Restaurant
}

// === GO OFFLINE ===
function goOffline(userId) {
  locationSocket?.emit('update_status', { riderId: userId, status: 'offline' });
}

// === DISCONNECT (call on logout) ===
function disconnectRider(userId) {
  goOffline(userId);
  ordersSocket?.disconnect();
  locationSocket?.disconnect();
}
```

### 8.2 Customer App Integration

```javascript
// ========================================
// CUSTOMER APP — COMPLETE SOCKET INTEGRATION
// ========================================

import { io } from 'socket.io-client';

const SERVER = 'https://drove-backend-f1d3892431b4.herokuapp.com';
let socket;

// === CONNECT (call after login or when viewing order) ===
function connectCustomer(userId, token) {
  socket = io(`${SERVER}/orders`, {
    transports: ['websocket', 'polling'],
    auth: { token },
    reconnection: true,
    reconnectionDelay: 1000,
  });

  socket.on('connect', () => {
    socket.emit('register_customer', { customerId: userId }, (res) => {
      console.log('Customer registered:', res);
    });
  });

  // --- LISTEN: Order status changes ---
  socket.on('order_status_updated', (data) => {
    // data.orderId (publicId), data.status, data.timestamp
    // + full order data when available
    updateOrderUI(data);

    // Status values:
    // PENDING         → "Waiting for payment confirmation"
    // CONFIRMED       → "Order confirmed!"
    // PREPARING       → "Restaurant is preparing your food"
    // READY_FOR_PICKUP→ "Food is ready, rider on the way"
    // ACCEPTED_BY_RIDER → "Rider accepted your order"
    // PICKED_UP       → "Rider picked up your food"
    // OUT_FOR_DELIVERY→ "Rider is on the way to you"
    // DELIVERED       → "Order delivered! Enjoy!"
  });

  // --- LISTEN: Rider assigned ---
  socket.on('rider_assigned', (data) => {
    // data.orderId, data.riderId
    showRiderAssigned();
    // Start showing map with rider location
  });

  // --- LISTEN: Rider GPS (live tracking) ---
  socket.on('rider_location_update', (data) => {
    // data.orderId
    // data.location.latitude
    // data.location.longitude
    // data.location.timestamp
    moveRiderMarkerOnMap(data.location.latitude, data.location.longitude);
  });
}

// === SUBSCRIBE TO SPECIFIC ORDER ===
function subscribeToOrder(orderId, userId) {
  socket?.emit('subscribe_to_order', { orderId, customerId: userId });
}

// === CLEANUP ===
function disconnectCustomer() {
  socket?.disconnect();
}
```

### 8.3 Restaurant/Shop Dashboard Integration

```javascript
// ========================================
// RESTAURANT DASHBOARD — COMPLETE SOCKET INTEGRATION
// ========================================

import { io } from 'socket.io-client';

const SERVER = 'https://drove-backend-f1d3892431b4.herokuapp.com';
let socket;

// === CONNECT (call after login) ===
function connectRestaurant(userId, vendorId, vendorType, token) {
  // vendorType = 'restaurant' or 'shop'
  socket = io(`${SERVER}/orders`, {
    transports: ['websocket', 'polling'],
    auth: { token },
    reconnection: true,
    reconnectionDelay: 1000,
  });

  socket.on('connect', () => {
    // Register as vendor to join the vendor room
    socket.emit('register_vendor', { vendorType, vendorId }, (res) => {
      console.log('Vendor registered:', res);
      // res.room = "vendor:restaurant:691e26ce..."
    });
  });

  // --- LISTEN: New order arrives ---
  socket.on('order_ready', (order) => {
    // Full order object with items, customer info, delivery info
    addOrderToList(order);
    playNotificationSound();
    showNotification(`New order ${order.publicId} - $${order.total}`);
  });

  // --- LISTEN: Rider accepted the order ---
  socket.on('rider_accepted', (data) => {
    // data includes full order + riderInfo
    // data.riderInfo = { id, name, phone, email, location, commissionPercent }
    updateOrderWithRider(data.publicId, data.riderInfo);
    showNotification(`Rider ${data.riderInfo?.name} accepted order ${data.publicId}`);
  });

  // --- LISTEN: Rider location (track rider coming to pick up) ---
  socket.on('rider_location_update', (data) => {
    // data.orderId, data.location.latitude, data.location.longitude
    updateRiderLocationOnOrder(data.orderId, data.location);
  });

  // --- LISTEN: Order status changes (e.g., PICKED_UP, DELIVERED) ---
  socket.on('order_status_updated', (data) => {
    // data.orderId, data.status, data.timestamp
    updateOrderStatus(data.orderId, data.status);
  });
}

// === START PREPARING (REST API) ===
async function startPreparing(orderId, token) {
  const res = await fetch(`${SERVER}/api/v1/orders/vendor/${orderId}/start-preparing`, {
    method: 'POST',
    headers: { Authorization: `Bearer ${token}` },
  });
  return res.json();
  // This triggers: new_order_nearby → riders, order_status_updated → customer
}

// === MARK PREPARED / READY (REST API) ===
async function markPrepared(orderId, token) {
  const res = await fetch(`${SERVER}/api/v1/orders/vendor/${orderId}/mark-prepared`, {
    method: 'POST',
    headers: { Authorization: `Bearer ${token}` },
  });
  return res.json();
  // This triggers: order_status_updated → customer + rider
}

// === DISCONNECT ===
function disconnectRestaurant() {
  socket?.disconnect();
}
```

---

## 9. Postman Testing Instructions

### How to Open Socket.IO in Postman

1. Open Postman
2. Click **"New"** button (top-left corner)
3. Select **"Socket.IO"** from the list
4. You'll see a URL bar with a "Connect" button

### Test 1: Restaurant Receives Orders

1. **URL**: `wss://drove-backend-f1d3892431b4.herokuapp.com/orders`
2. Click **Settings** tab under URL → Client version: **v4**
3. Click **Connect** → Status shows ✅ Connected

4. **Send registration:**
   - Bottom bar → Event name: `register_vendor`
   - Message tab → select **JSON**
   - Type: `{"vendorType":"restaurant","vendorId":"691e26ce1d88bac5205b0bdb"}`
   - Toggle **Ack** to ON
   - Click **Send**
   - ✅ See response: `{"success":true,"room":"vendor:restaurant:691e26ce1d88bac5205b0bdb"}`

5. **Add listeners:**
   - Click **Events** tab (top, next to Messages)
   - Click **+ Add** 
   - Type `order_ready` → toggle ON
   - Click **+ Add** → `rider_accepted` → toggle ON
   - Click **+ Add** → `rider_location_update` → toggle ON

6. Now create an order from the app. You'll see `order_ready` appear in the Events tab!

### Test 2: Rider Goes Online & Gets Orders

**Open 2 Postman Socket.IO tabs:**

**Tab A** — `/orders`:
1. URL: `wss://drove-backend-f1d3892431b4.herokuapp.com/orders` → Connect
2. Send `register_rider` with `{"riderId":"690ba26be8e0b05bd9f1d8a0"}` (Ack ON)
3. Add listener: `new_order_nearby`, `order_accepted`, `order_no_longer_available`

**Tab B** — `/rider-location`:
1. URL: `wss://drove-backend-f1d3892431b4.herokuapp.com/rider-location` → Connect
2. Send `register_rider` with `{"riderId":"690ba26be8e0b05bd9f1d8a0"}` (Ack ON)
3. Send `update_status` with `{"riderId":"690ba26be8e0b05bd9f1d8a0","status":"online"}` (Ack ON)
4. Send `update_location` with `{"riderId":"690ba26be8e0b05bd9f1d8a0","latitude":28.302646,"longitude":70.126787,"speed":0,"heading":0,"accuracy":10}` (Ack ON)
5. Add listeners: `rider_status_updated`, `rider_location_updated`

Now the rider is online and located. Create an order nearby → Tab A will receive `new_order_nearby`!

### Test 3: Customer Tracks Order

1. URL: `wss://drove-backend-f1d3892431b4.herokuapp.com/orders` → Connect
2. Send `register_customer` with `{"customerId":"68dbbd4e7ba63e26ddb734ce"}` (Ack ON)
3. Add listeners: `order_status_updated`, `rider_assigned`, `rider_location_update`
4. Place an order → watch events arrive as status progresses!

### Test 4: Ping/Pong Health Check

1. Connect to any namespace
2. Send `ping` with `{}` (Ack ON)
3. ✅ Get ack: `{"success":true}`
4. ✅ See `pong` event in Events tab

### Test 5: Find Nearby Riders

1. Connect to `/rider-location`
2. Send `find_nearby_riders` with `{"latitude":28.302646,"longitude":70.126787,"radiusKm":50,"limit":10}` (Ack ON)
3. ✅ Get list of riders with distance

---

## 10. Troubleshooting & FAQ

### Q: I connected but I'm not receiving any events?
**A:** You must REGISTER after connecting. Just connecting is not enough. Send `register_customer`, `register_rider`, or `register_vendor` immediately after `connect` event fires.

### Q: I was receiving events but stopped after a while?
**A:** Socket probably reconnected silently. Add a `connect` event listener that re-registers your role every time.

### Q: Customer is not receiving `rider_location_update`?
**A:** This only works when:
1. Customer is registered with their correct `userId` (from login)
2. A rider has an active order for this customer (status: PREPARING, READY_FOR_PICKUP, ACCEPTED_BY_RIDER, PICKED_UP, or OUT_FOR_DELIVERY)
3. The rider is actually sending `update_location` on `/rider-location`

### Q: Rider is not receiving `new_order_nearby`?
**A:** Check all of these:
1. Rider is registered on `/orders` namespace (not just `/rider-location`)
2. Rider status is `ONLINE` (not OFFLINE or BUSY)
3. Rider is approved (`isApproved: true` in database)
4. Rider's location is within 50km of the order's delivery address
5. Rider's city matches the order's city
6. Rider hasn't already rejected this order
7. For CASH orders: rider needs sufficient wallet balance

### Q: What's the difference between `order_status_updated` and `order_status_update`?
**A:** 
- `order_status_updated` → sent to **Customer**
- `order_status_update` → sent to **Rider** (for their active delivery)
- Different events, different recipients, same purpose (notify about status change)

### Q: What's the difference between `order_accepted` and `order_no_longer_available`?
**A:** They go to DIFFERENT riders for the same order:
- `order_accepted` → sent to **the rider who accepted** (your acceptance confirmed)
- `order_no_longer_available` → sent to **all other notified riders** (someone else got it, remove from your list)

### Q: What's the difference between `rider_location_updated` and `rider_location_update`?
**A:** Both on `/rider-location` namespace:
- `rider_location_updated` → broadcast to **ALL** clients (for admin live map showing all riders)
- `rider_location_update` → sent only to **admins who subscribed** to a specific rider via `admin_subscribe_rider`

On `/orders` namespace there's also a `rider_location_update` → sent to the **customer** and **vendor** for active orders.

### Q: Do I need authentication (JWT token) for WebSocket?
**A:** Currently the token is sent but NOT enforced on WebSocket connections. However, send it anyway (via `auth.token` in connection options) as enforcement may be added in the future.

### Q: How do I test without a real GPS?
**A:** Just send fake coordinates via `update_location`:
```json
{"riderId":"690ba26be8e0b05bd9f1d8a0","latitude":28.302646,"longitude":70.126787,"speed":0,"heading":0,"accuracy":10}
```
Change lat/lng slightly each time to simulate movement.

### Q: What happens when rider's app goes to background?
**A:** Socket will disconnect. Server waits 5 seconds (grace period). If rider reconnects within 5s, they stay online. Otherwise marked offline. Mobile apps should use background location services to keep the socket alive during active deliveries.

### Q: Can a rider be on both `/orders` and `/rider-location` simultaneously?
**A:** Yes, and they MUST be. These are 2 independent connections:
- `/orders` → for receiving order notifications (`new_order_nearby`, `order_accepted`, etc.)
- `/rider-location` → for sending GPS and managing online/offline status

### Q: What order statuses exist?
**A:**
| Status | Meaning | Set By |
|---|---|---|
| `PENDING` | Draft confirmed, waiting for payment | System |
| `CONFIRMED` | Payment verified | System/Admin |
| `PREPARING` | Vendor started cooking | Vendor (start-preparing API) |
| `READY_FOR_PICKUP` | Food is ready | Vendor (mark-prepared API) |
| `ACCEPTED_BY_RIDER` | Rider accepted the order | System (on rider accept) |
| `PICKED_UP` | Rider has the food | Rider (or vendor for pickup orders) |
| `OUT_FOR_DELIVERY` | Rider heading to customer | System |
| `DELIVERED` | Order complete | Rider (delivered API) |
| `CANCELLED` | Order cancelled | System/Admin |
