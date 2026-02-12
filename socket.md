# Drovi Real-Time Socket System — Mobile Developer Guide

> **Complete reference for implementing WebSocket connections, real-time order notifications, rider location tracking, and status management in Drovi mobile apps.**

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Socket Namespaces & Connection Details](#2-socket-namespaces--connection-details)
3. [Customer Flow](#3-customer-flow)
4. [Rider Flow](#4-rider-flow)
5. [Vendor (Restaurant/Shop) Flow](#5-vendor-restaurantshop-flow)
6. [Admin Flow](#6-admin-flow)
7. [Complete Event Reference](#7-complete-event-reference)
8. [Redis & Location Architecture](#8-redis--location-architecture)
9. [Order Lifecycle & Notification Flow](#9-order-lifecycle--notification-flow)
10. [Error Handling & Reconnection](#10-error-handling--reconnection)
11. [Implementation Checklist](#11-implementation-checklist)

---

## 1. Architecture Overview

Drovi uses **two Socket.IO namespaces** running on the same backend server:

| Namespace | Purpose | Used By |
|---|---|---|
| `/orders` | Order notifications, status updates, payment alerts | All roles |
| `/rider-location` | GPS tracking, rider online/offline status | Riders, Admins |

**Backend Stack:**
- NestJS WebSocket Gateways (`@WebSocketGateway`)
- Socket.IO server with WebSocket + polling fallback
- Redis (Upstash) for geospatial rider tracking, status metadata, pub/sub
- PostgreSQL (Prisma) for persistent order/rider data

```
┌─────────────┐     ┌──────────────────────────────────────────────┐
│  Customer    │────▶│                                              │
│  Mobile App  │◀────│         BACKEND SERVER                       │
└─────────────┘     │                                              │
                    │   ┌──────────────┐  ┌─────────────────────┐  │
┌─────────────┐     │   │ /orders      │  │ /rider-location     │  │
│  Rider       │────▶│   │ namespace    │  │ namespace           │  │
│  Mobile App  │◀────│   │              │  │                     │  │
└─────────────┘     │   │ - Order      │  │ - GPS updates       │  │
                    │   │   notifs     │  │ - Status mgmt       │  │
┌─────────────┐     │   │ - Status     │  │ - Nearby riders     │  │
│  Vendor      │────▶│   │   updates   │  │ - Location broadcast│  │
│  Dashboard   │◀────│   └──────┬───────┘  └──────────┬──────────┘  │
└─────────────┘     │          │                      │             │
                    │          ▼                      ▼             │
┌─────────────┐     │   ┌──────────────┐  ┌──────────────────────┐ │
│  Admin       │────▶│   │  PostgreSQL  │  │       Redis          │ │
│  Dashboard   │◀────│   │  (Prisma)    │  │  - Geo index         │ │
└─────────────┘     │   │              │  │  - Status hashes     │ │
                    │   └──────────────┘  │  - Location history  │ │
                    │                     │  - Pub/Sub           │ │
                    │                     └──────────────────────┘ │
                    └──────────────────────────────────────────────┘
```

---

## 2. Socket Namespaces & Connection Details

### Connection URL

```
Base URL: https://drove-backend-f1d3892431b4.herokuapp.com
```

### Namespace: `/orders`

```javascript
const socket = io("https://drove-backend-f1d3892431b4.herokuapp.com/orders", {
  transports: ["websocket", "polling"],
  auth: { token: "<accessToken>" },
  reconnection: true,
  reconnectionAttempts: Infinity,
  reconnectionDelay: 1000,
  reconnectionDelayMax: 5000,
  timeout: 20000,
});
```

### Namespace: `/rider-location`

```javascript
const socket = io("wss://drove-backend-f1d3892431b4.herokuapp.com/rider-location", {
  transports: ["websocket", "polling"],
  auth: { token: "<accessToken>" },
  reconnection: true,
  reconnectionDelay: 1000,
  reconnectionAttempts: 5,
});
```

> **Note:** The `/rider-location` namespace is **only needed for the Rider app** and the Admin tracking dashboard.

---

## 3. Customer Flow

### 3.1 Connection & Registration

```
Customer App Start
  │
  ├─ Connect to /orders namespace
  │
  ├─ On "connect" → emit "register_customer"
  │     Payload: { customerId: "<user.id>" }
  │     Response: { success: true, customerId: "..." }
  │
  └─ Start heartbeat (ping every 25s)
```

**Code Example:**
```javascript
// Connect
socket = io(`https://drove-backend-f1d3892431b4.herokuapp.com/orders`, {
  transports: ["websocket", "polling"],
  auth: { token: accessToken },
  reconnection: true,
});

// Register on connect
socket.on("connect", () => {
  socket.emit("register_customer", { customerId: userId }, (response) => {
    if (response.success) {
      console.log("Customer registered");
    }
  });
});
```

### 3.2 Subscribe to Order Updates

After placing an order, subscribe to get real-time status updates:

```javascript
// Subscribe to a specific order
socket.emit("subscribe_to_order", {
  orderId: "<orderId>",
  customerId: "<userId>"
}, (response) => {
  if (response.success) {
    console.log("Subscribed to order updates");
  }
});

// Unsubscribe when done (e.g., order delivered)
socket.emit("unsubscribe_from_order", { orderId: "<orderId>" });
```

### 3.3 Events the Customer Listens To

| Event | Payload | When |
|---|---|---|
| `order_status_updated` | `{ orderId, status, timestamp, ...orderData }` | Any order status change (CONFIRMED → PREPARING → READY → PICKED_UP → DELIVERED) |
| `rider_assigned` | `{ orderId, riderId }` | A rider accepts the delivery |
| `rider_location_update` | `{ orderId, location: { latitude, longitude, timestamp } }` | Rider GPS update while delivering (every few seconds) |

**Code Example:**
```javascript
// Listen for status changes
socket.on("order_status_updated", (data) => {
  // data.status: "CONFIRMED" | "PREPARING" | "READY_FOR_PICKUP" |
  //              "ACCEPTED_BY_RIDER" | "PICKED_UP" | "OUT_FOR_DELIVERY" | "DELIVERED"
  updateOrderUI(data.orderId, data.status);
});

// Listen for rider assignment
socket.on("rider_assigned", (data) => {
  showRiderAssigned(data.orderId, data.riderId);
});

// Listen for rider location (show on map)
socket.on("rider_location_update", (data) => {
  updateRiderMarkerOnMap(data.orderId, data.location.latitude, data.location.longitude);
});
```

### 3.4 Multi-Order Support

The customer can subscribe to **multiple orders** simultaneously. Each event includes the `orderId` field so you can match updates to the correct order in your UI.

```javascript
// Subscribe to multiple active orders
activeOrders.forEach(order => {
  socket.emit("subscribe_to_order", {
    orderId: order.id,
    customerId: userId,
  });
});

// Handle updates — filter by orderId
socket.on("order_status_updated", (data) => {
  const orderIndex = activeOrders.findIndex(o => o.id === data.orderId);
  if (orderIndex >= 0) {
    activeOrders[orderIndex].status = data.status;
    refreshOrderList();
  }
});
```

---

## 4. Rider Flow

The rider uses **both** socket namespaces:
- `/orders` — for receiving order notifications and status updates
- `/rider-location` — for GPS tracking and online/offline status

### 4.1 Complete Startup Sequence

```
Rider App Login
  │
  ├─ 1. Request Location Permission (GPS)
  │
  ├─ 2. Connect to /rider-location namespace
  │     └─ emit "register_rider" → { riderId: user.id }
  │        Response: { success: true, status: "online" }
  │        ✅ Rider is now ONLINE in Redis
  │
  ├─ 3. Set status ONLINE via REST API
  │     └─ POST /riders/go-online
  │        ✅ Database status = ONLINE, isAvailable = true
  │
  ├─ 4. Start GPS tracking
  │     └─ Use device GPS (watchPosition)
  │     └─ Every position change → emit "update_location"
  │
  ├─ 5. Connect to /orders namespace
  │     └─ emit "register_rider" → { riderId: user.id }
  │        ✅ Rider can now receive order notifications
  │
  └─ 6. Start periodic status sync (every 30 seconds)
        └─ POST /riders/go-online  (keeps DB in sync)
        └─ emit "update_status" → { riderId, status: "online" }
```

### 4.2 Setting Rider Online / Offline

**Going ONLINE:**
```javascript
// 1. Connect to location socket
await locationSocket.connect(riderId, token);

// 2. Register rider (auto-marks online in Redis)
locationSocket.emit("register_rider", { riderId: userId }, (response) => {
  // response.status = "online"
});

// 3. Update database via REST
await fetch("/riders/go-online", { method: "POST" });

// 4. Update socket status
locationSocket.emit("update_status", { riderId: userId, status: "online" }, (response) => {
  // response.success = true
});

// 5. Start GPS tracking
startGPSTracking();
```

**Going OFFLINE:**
```javascript
// 1. Update socket status
locationSocket.emit("update_status", { riderId: userId, status: "offline" }, (response) => {
  // response.success = true
});

// 2. Stop GPS tracking
stopGPSTracking();

// 3. Update database via REST
await fetch("/riders/go-offline", { method: "POST" });

// 4. Disconnect location socket
locationSocket.disconnect();
```

**Status Values:** `"online"` | `"offline"` | `"busy"`

> **Important:** When the rider socket disconnects unexpectedly, the backend has a **5-second grace period** before marking the rider offline. If the rider reconnects within 5 seconds, they stay online.

### 4.3 GPS Location Updates

```javascript
// Emit location to /rider-location namespace
locationSocket.emit("update_location", {
  riderId: "<user.id>",
  latitude: 2.0469,
  longitude: 45.3182,
  heading: 180,        // degrees, optional
  speed: 12.5,         // m/s, optional
  accuracy: 15,        // meters, optional
}, (response) => {
  // response.success = true, response.timestamp = 1234567890
});
```

**What happens server-side on each `update_location`:**
1. Location stored in Redis geospatial index (`GEOADD riders:locations`)
2. Metadata stored in Redis hash (`rider:status:{riderId}`) with 5-min TTL
3. Location added to history sorted set (`rider:history:{riderId}`, last 100)
4. Rider availability ranking updated
5. **Broadcast to all admin dashboards** via `rider_location_updated`
6. **Broadcast to customers & vendors** of active orders via `rider_location_update` (through the `/orders` namespace)

### 4.4 Receiving Order Notifications

Listen on the **`/orders`** namespace:

```javascript
// New order available nearby
ordersSocket.on("new_order_nearby", (data) => {
  // data = {
  //   orderId: "...",
  //   publicId: "DRV-XXXX",
  //   distance: 2.5,                    // km from rider
  //   pickupAddress: "...",
  //   deliveryAddress: "...",
  //   estimatedEarnings: 150,            // rider commission amount
  //   deliveryFee: 200,
  //   commissionPercent: 75,
  //   paymentMethod: "CASH" | "EVC_PLUS",
  //   items: [...],
  //   vendorName: "...",
  //   vendorType: "restaurant" | "shop",
  //   timestamp: 1234567890
  // }
  showOrderNotificationModal(data);
});

// Order no longer available (another rider accepted it)
ordersSocket.on("order_no_longer_available", (data) => {
  // data = { orderId, reason: "accepted_by_another_rider", message: "..." }
  removeOrderFromList(data.orderId);
});

// Order status update (e.g., vendor marked as ready for pickup)
ordersSocket.on("order_status_update", (data) => {
  // data = { orderId, status, publicId, timestamp }
  updateActiveOrder(data.orderId, data.status);
});

// Order accepted confirmation (after rider calls accept API)
ordersSocket.on("order_accepted", (data) => {
  // data = { orderId, order: {...}, timestamp }
  addToActiveDeliveries(data.order);
});
```

### 4.5 Accepting / Rejecting Orders

Done via **REST API** (not socket):

```javascript
// Accept order
const response = await fetch(`/orders/${orderId}/rider-accept`, {
  method: "PATCH",
  headers: { Authorization: `Bearer ${token}` },
});
// Success → rider status changes to BUSY, order assigned

// Reject order
const response = await fetch(`/orders/${orderId}/rider-reject`, {
  method: "PATCH",
  headers: { Authorization: `Bearer ${token}` },
  body: JSON.stringify({ reason: "Too far" }),
});
```

### 4.6 Multi-Order Modal

The rider can receive **multiple order notifications simultaneously**. The app should:
1. Queue incoming `new_order_nearby` events
2. Display them in a scrollable modal/list
3. Remove orders when `order_no_longer_available` fires
4. De-duplicate by `orderId` (same order can be re-notified on retries)

```javascript
const pendingOrders = [];

ordersSocket.on("new_order_nearby", (order) => {
  // De-duplicate
  if (!pendingOrders.find(o => o.orderId === order.orderId)) {
    pendingOrders.push(order);
    showOrderModal();
  }
});

ordersSocket.on("order_no_longer_available", (data) => {
  pendingOrders = pendingOrders.filter(o => o.orderId !== data.orderId);
  if (pendingOrders.length === 0) hideOrderModal();
});
```

### 4.7 Connection Status Events

Listen on `/rider-location` namespace:

```javascript
// Status confirmed by backend
locationSocket.on("rider_status_updated", (data) => {
  // data = { riderId, status: "online" | "offline" | "busy" }
  updateStatusIndicator(data.status);
});
```

---

## 5. Vendor (Restaurant/Shop) Flow

### 5.1 Connection & Registration

Vendors connect to the **`/orders`** namespace and join a **room** based on their vendor type and ID:

```javascript
// Connect
socket = io(`https://drove-backend-f1d3892431b4.herokuapp.com/orders`, {
  transports: ["websocket", "polling"],
  auth: { token: accessToken },
  reconnection: true,
});

// Register vendor room on connect
socket.on("connect", () => {
  socket.emit("register_vendor", {
    vendorType: "restaurant",      // or "shop"
    vendorId: "<restaurant.id>",   // the profile ID (not user ID)
  }, (response) => {
    // response = { success: true, room: "vendor:restaurant:abc123" }
  });
});
```

> **Important:** The `vendorId` is the **profile ID** (`user.restaurant.id` or `user.shop.id`), NOT the `user.id`.

### 5.2 Events the Vendor Listens To

| Event | Payload | When |
|---|---|---|
| `order_ready` | Full order object (see below) | New confirmed order for this vendor |
| `rider_accepted` | Order object with `riderInfo` | A rider accepted the delivery |
| `rider_location_update` | `{ orderId, location: { latitude, longitude, timestamp } }` | Rider GPS update (for tracking rider approaching) |

### 5.3 New Order Notification

```javascript
socket.on("order_ready", (order) => {
  // Only process CONFIRMED orders for the bell notification
  if (order.status === "CONFIRMED") {
    playNotificationSound();
    showOrderModal(order);
  }

  // order = {
  //   id: "...",
  //   publicId: "DRV-1234",
  //   status: "CONFIRMED",
  //   items: [...],
  //   total: 450,
  //   deliveryFee: 100,
  //   paymentMethod: "CASH" | "EVC_PLUS",
  //   delivery: { address, latitude, longitude },
  //   customer: { firstName, lastName, phone },
  //   createdAt: "...",
  // }
});
```

### 5.4 Rider Accepted Notification

```javascript
socket.on("rider_accepted", (order) => {
  // order includes riderInfo:
  // order.riderInfo = {
  //   name: "Ahmed Ali",
  //   phone: "+252...",
  //   vehicleType: "motorcycle",
  //   avatar: "...",
  // }
  showRiderInfo(order.publicId, order.riderInfo);
});
```

### 5.5 Rider Location Tracking

```javascript
socket.on("rider_location_update", (data) => {
  // Track rider approaching the restaurant
  updateRiderOnMap(data.orderId, data.location.latitude, data.location.longitude);
});
```

### 5.6 Order Status Management (REST API)

```javascript
// Start preparing
await fetch(`/orders/${orderId}/start-preparing`, { method: "PATCH" });

// Mark as prepared (ready for pickup)
await fetch(`/orders/${orderId}/mark-prepared`, { method: "PATCH" });
```

---

## 6. Admin Flow

### 6.1 Connection & Registration

```javascript
// Connect to /orders namespace
ordersSocket = io(`https://drove-backend-f1d3892431b4.herokuapp.com/orders`, {
  transports: ["websocket", "polling"],
  auth: { token: accessToken },
});

ordersSocket.on("connect", () => {
  ordersSocket.emit("register_admin", { userId: "<user.id>" }, (response) => {
    // response = { success: true, adminId: "..." }
    // Backend auto-loads admin's assignedCity for city-filtered notifications
  });
});

// Connect to /rider-location namespace (for tracking dashboard)
locationSocket = io(`https://drove-backend-f1d3892431b4.herokuapp.com/rider-location`, {
  transports: ["websocket", "polling"],
  auth: { token: accessToken },
});
```

### 6.2 Events the Admin Listens To

**On `/orders` namespace:**

| Event | Payload | When |
|---|---|---|
| `new_payment` | Order object | New EVC Plus payment needs verification |
| `payment_verified` | `{ orderId, adminId }` | Another admin verified a payment |
| `payment_rejected` | `{ orderId, adminId, reason }` | Another admin rejected a payment |
| `order_needs_attention` | `{ orderId, reason, timestamp }` | No riders available for an order |
| `order_escalation` | Detailed escalation object | Order unassigned for 10+ minutes (HIGH priority) |
| `new_deposit_request` | Deposit data | Rider submitted a deposit |
| `new_withdrawal_request` | Withdrawal data | Rider requested a withdrawal |

> **City Filtering:** All admin notifications are filtered by `assignedCity`. An admin in "Mogadishu" only receives notifications for Mogadishu orders.

**On `/rider-location` namespace:**

| Event | Payload | When |
|---|---|---|
| `rider_location_updated` | `{ riderId, latitude, longitude, timestamp, status, heading, speed, accuracy }` | Any rider sends GPS update (broadcast to all) |
| `rider_status_updated` | `{ riderId, status }` | Rider goes online/offline/busy |

### 6.3 Admin Subscribe to Specific Rider

```javascript
// Subscribe to detailed location updates for one rider
locationSocket.emit("admin_subscribe_rider", { riderId: "<userId>" }, (response) => {
  // response = { success: true }
});

// Now receive detailed updates just for that rider
locationSocket.on("rider_location_update", (location) => {
  // Full RiderLocation object
});
```

---

## 7. Complete Event Reference

### Events YOU EMIT (Client → Server)

| Namespace | Event | Payload | Role | Description |
|---|---|---|---|---|
| `/orders` | `register_admin` | `{ userId }` | Admin | Register for admin notifications |
| `/orders` | `register_customer` | `{ customerId }` | Customer | Register for customer notifications |
| `/orders` | `register_rider` | `{ riderId }` | Rider | Register for rider order notifications |
| `/orders` | `register_vendor` | `{ vendorType, vendorId }` | Vendor | Join vendor notification room |
| `/orders` | `subscribe_to_order` | `{ orderId, customerId }` | Customer | Subscribe to specific order updates |
| `/orders` | `unsubscribe_from_order` | `{ orderId }` | Customer | Stop tracking an order |
| `/orders` | `ping` | (none) | All | Heartbeat keep-alive |
| `/rider-location` | `register_rider` | `{ riderId }` | Rider | Register + auto set ONLINE |
| `/rider-location` | `update_location` | `{ riderId, latitude, longitude, heading?, speed?, accuracy? }` | Rider | Send GPS position |
| `/rider-location` | `update_status` | `{ riderId, status }` | Rider | Set online/offline/busy |
| `/rider-location` | `find_nearby_riders` | `{ latitude, longitude, radiusKm?, limit? }` | Admin | Query nearby riders |
| `/rider-location` | `get_rider_location` | `{ riderId }` | Admin | Get one rider's position |
| `/rider-location` | `admin_subscribe_rider` | `{ riderId }` | Admin | Subscribe to one rider's updates |

### Events YOU LISTEN TO (Server → Client)

| Namespace | Event | Payload | Role | Description |
|---|---|---|---|---|
| `/orders` | `order_ready` | Full order object | Vendor | New confirmed order |
| `/orders` | `rider_accepted` | Order + riderInfo | Vendor | Rider picked up the job |
| `/orders` | `rider_location_update` | `{ orderId, location }` | Customer, Vendor | Rider GPS during delivery |
| `/orders` | `order_status_updated` | `{ orderId, status, timestamp, ...order }` | Customer | Order status changed |
| `/orders` | `rider_assigned` | `{ orderId, riderId }` | Customer | Rider assigned to order |
| `/orders` | `new_order_nearby` | Order notification data | Rider | New delivery opportunity |
| `/orders` | `order_no_longer_available` | `{ orderId, reason, message }` | Rider | Someone else took the order |
| `/orders` | `order_status_update` | `{ orderId, status, timestamp }` | Rider | Active order status change |
| `/orders` | `order_accepted` | `{ orderId, order, timestamp }` | Rider | Confirmation of acceptance |
| `/orders` | `new_payment` | Order object | Admin | EVC payment to verify |
| `/orders` | `payment_verified` | `{ orderId, adminId }` | Admin | Payment verified |
| `/orders` | `payment_rejected` | `{ orderId, adminId, reason }` | Admin | Payment rejected |
| `/orders` | `order_needs_attention` | `{ orderId, reason, timestamp }` | Admin | No riders available |
| `/orders` | `order_escalation` | Escalation data | Admin | 10-min unassigned order |
| `/orders` | `new_deposit_request` | Deposit data | Admin | Rider deposit request |
| `/orders` | `new_withdrawal_request` | Withdrawal data | Admin | Rider withdrawal request |
| `/orders` | `pong` | (none) | All | Heartbeat response |
| `/rider-location` | `rider_location_updated` | `{ riderId, lat, lng, timestamp, status, heading, speed, accuracy }` | Admin | Any rider moved |
| `/rider-location` | `rider_status_updated` | `{ riderId, status }` | Rider, Admin | Status confirmed |

---

## 8. Redis & Location Architecture

### 8.1 Redis Keys

| Key Pattern | Type | Purpose | TTL |
|---|---|---|---|
| `riders:locations` | Geo Set | Geospatial index of all rider positions | Persistent |
| `rider:status:{userId}` | Hash | Status metadata (lat, lng, timestamp, status, heading, speed, accuracy) | 5 min |
| `rider:history:{userId}` | Sorted Set | Last 100 location points for analytics | 24 hours |
| `riders:availability` | Sorted Set | Rider availability ranking by last active timestamp | Persistent |

### 8.2 Location Update Pipeline

```
Rider GPS → emit "update_location"
  │
  ├─ Redis GEOADD riders:locations (longitude, latitude, riderId)
  ├─ Redis HSET rider:status:{id} (all metadata fields)
  ├─ Redis EXPIRE rider:status:{id} 300 (5 min TTL)
  ├─ Redis ZADD riders:availability (timestamp, riderId)
  ├─ Redis ZADD rider:history:{id} (timestamp, JSON location)
  │
  ├─ Broadcast "rider_location_updated" to ALL on /rider-location
  ├─ Send to admin subscribers of this rider
  │
  └─ Query active orders for this rider (Prisma)
       └─ For each active order:
            ├─ Emit "rider_location_update" to customer socket (/orders)
            └─ Emit "rider_location_update" to vendor room (/orders)
```

### 8.3 Stale Cleanup (Every 2 Minutes)

The backend runs a **cron job every 2 minutes** that:
1. Checks all riders in Redis for stale metadata (no update in 5 min)
2. Marks stale riders as `offline`
3. Cross-checks DB: riders marked ONLINE/BUSY in DB but missing from Redis → marked OFFLINE

---

## 9. Order Lifecycle & Notification Flow

### 9.1 Complete Order Flow

```
Customer Places Order
  │
  ├─ Order created (status: PENDING)
  ├─ Payment verified (status: CONFIRMED)
  │     └─ Vendor notified via "order_ready" socket event
  │
  ▼
Vendor Clicks "Start Preparing"
  │
  ├─ Order status → PREPARING
  ├─ Customer notified via "order_status_updated"
  │
  ├─ ⚡ IMMEDIATELY: Find top 4 nearest ONLINE riders
  │     └─ Redis GEORADIUS query (50km radius)
  │     └─ Filter: isApproved=true, isAvailable=true, status=ONLINE
  │     └─ Exclude riders who already rejected this order
  │     └─ For CASH orders: check rider wallet balance ≥ order total
  │     └─ Notify via "new_order_nearby" to each rider's socket
  │
  ├─ ⏰ RETRY: Every 2 minutes for 10 minutes total (5 retries)
  │     └─ Re-query nearest riders (picks up newly online riders)
  │     └─ Re-notify (same order, riders de-duplicate on client)
  │
  └─ 🚨 ESCALATION: After 10 minutes with no rider
        └─ "order_needs_attention" / "order_escalation" sent to admins
        └─ Filtered by admin's assignedCity matching order city
  │
  ▼
Rider Accepts Order (REST: PATCH /orders/:id/rider-accept)
  │
  ├─ Order status → ACCEPTED_BY_RIDER
  ├─ Rider status → BUSY, currentOrderId set
  ├─ Rider earnings record created
  ├─ Retry notifications stopped
  ├─ Other notified riders receive "order_no_longer_available"
  ├─ Customer notified: "order_status_updated" + "rider_assigned"
  ├─ Vendor notified: "rider_accepted" (with riderInfo)
  │
  ▼
Vendor Marks "Prepared" (REST: PATCH /orders/:id/mark-prepared)
  │
  ├─ Order status → READY_FOR_PICKUP
  ├─ Customer notified: "order_status_updated"
  ├─ Rider notified: "order_status_update" (status: READY_FOR_PICKUP)
  │
  ▼
Rider Picks Up Order (REST: PATCH /orders/:id/picked-up)
  │
  ├─ Order status → PICKED_UP / OUT_FOR_DELIVERY
  ├─ Customer notified: "order_status_updated"
  │
  ▼ (Rider GPS continuously broadcasts to customer & vendor)
  │
  ▼
Rider Delivers Order (REST: PATCH /orders/:id/delivered)
  │
  ├─ Order status → DELIVERED
  ├─ Customer notified: "order_status_updated"
  ├─ Rider status → ONLINE (back to available)
  └─ Rider currentOrderId cleared
```

### 9.2 Rider Notification Data Format

```json
{
  "orderId": "clxyz123...",
  "publicId": "DRV-0042",
  "distance": 2.3,
  "estimatedEarnings": 112.5,
  "deliveryFee": 150,
  "commissionPercent": 75,
  "paymentMethod": "CASH",
  "vendorName": "Burger Palace",
  "vendorType": "restaurant",
  "pickupAddress": "123 Main St, Mogadishu",
  "pickupLocation": { "latitude": 2.0469, "longitude": 45.3182 },
  "deliveryAddress": "456 Oak Ave, Mogadishu",
  "deliveryLocation": { "latitude": 2.0500, "longitude": 45.3200 },
  "deliveryDistance": 3.5,
  "items": [
    { "name": "Chicken Burger", "quantity": 2, "price": 200 }
  ],
  "total": 500,
  "timestamp": 1700000000000
}
```

---

## 10. Error Handling & Reconnection

### 10.1 Reconnection Strategy

| Parameter | Customer | Rider | Vendor |
|---|---|---|---|
| `reconnection` | `true` | `true` | `true` |
| `reconnectionAttempts` | `Infinity` | `5` (location) / `Infinity` (orders) | `Infinity` |
| `reconnectionDelay` | `1000ms` | `1000ms` | `1000ms` |
| `reconnectionDelayMax` | `5000ms` | `5000ms` | `5000ms` |

### 10.2 Re-registration on Reconnect

**Critical:** After every reconnection, you **MUST re-register**:

```javascript
socket.on("connect", () => {
  // Re-register based on role
  socket.emit("register_customer", { customerId: userId });
  // or
  socket.emit("register_rider", { riderId: userId });
  // or
  socket.emit("register_vendor", { vendorType: "restaurant", vendorId: profileId });
  // or
  socket.emit("register_admin", { userId: userId });

  // Re-subscribe to active orders (customer)
  activeOrders.forEach(order => {
    socket.emit("subscribe_to_order", { orderId: order.id, customerId: userId });
  });
});
```

### 10.3 Rider Disconnect Grace Period

When a rider's `/rider-location` socket disconnects:
- Backend waits **5 seconds** before marking offline
- If rider reconnects within 5 seconds → stays online, no disruption
- If not reconnected → `updateRiderStatus(riderId, "offline")` called

### 10.4 Heartbeat (Keep-Alive)

Send a `ping` every 25 seconds on the `/orders` namespace:

```javascript
setInterval(() => {
  if (socket.connected) {
    socket.emit("ping");
  }
}, 25000);

socket.on("pong", () => {
  // Connection is alive
});
```

### 10.5 Vendor Registration Retry

Vendor registration uses **exponential backoff** if it fails:
- Retry delays: 500ms, 1000ms, 2000ms, 4000ms, 5000ms (max)
- Max 5 retries

---

## 11. Implementation Checklist

### Customer App

- [ ] Connect to `/orders` namespace on app start (when logged in)
- [ ] Emit `register_customer` on every `connect` event
- [ ] After placing order: `subscribe_to_order`
- [ ] Listen to `order_status_updated` → update order status UI
- [ ] Listen to `rider_assigned` → show rider info
- [ ] Listen to `rider_location_update` → show rider on map
- [ ] Handle `disconnect` → show "reconnecting" indicator
- [ ] Re-register + re-subscribe on reconnect
- [ ] `unsubscribe_from_order` when order completed
- [ ] Support multiple simultaneous order subscriptions
- [ ] Heartbeat ping every 25 seconds

### Rider App

- [ ] On login: request GPS permission
- [ ] Connect to `/rider-location` → `register_rider` (auto goes ONLINE)
- [ ] REST: `POST /riders/go-online` (database sync)
- [ ] Start GPS tracking → emit `update_location` on every position change
- [ ] Connect to `/orders` → `register_rider`
- [ ] Listen to `new_order_nearby` → show in notification modal
- [ ] De-duplicate incoming orders by `orderId`
- [ ] Listen to `order_no_longer_available` → remove from list
- [ ] Listen to `order_status_update` → update active delivery UI
- [ ] Listen to `order_accepted` → add to active deliveries
- [ ] Accept/Reject via REST API (`PATCH /orders/:id/rider-accept` or `rider-reject`)
- [ ] Handle multi-order notification queue
- [ ] Going OFFLINE: `update_status` → stop GPS → REST go-offline → disconnect
- [ ] Periodic status sync every 30 seconds
- [ ] Handle disconnect grace period (5 seconds)
- [ ] Re-register on both namespaces on reconnect
- [ ] Listen to `rider_status_updated` for status confirmation

### Vendor App

- [ ] Connect to `/orders` namespace on dashboard open
- [ ] Emit `register_vendor` with correct `vendorType` and profile `vendorId`
- [ ] Listen to `order_ready` → show new order notification (only for CONFIRMED status)
- [ ] Listen to `rider_accepted` → show rider info on order
- [ ] Listen to `rider_location_update` → track rider on map
- [ ] Handle order actions via REST (start-preparing, mark-prepared)
- [ ] Re-register vendor room on reconnect
- [ ] Show online/offline connection indicator
- [ ] Vendor registration retry with exponential backoff

### Admin App

- [ ] Connect to `/orders` namespace → `register_admin`
- [ ] Connect to `/rider-location` namespace (for tracking)
- [ ] Listen to `new_payment` → show payment verification modal
- [ ] Listen to `order_needs_attention` / `order_escalation` → show alerts
- [ ] Listen to `new_deposit_request` / `new_withdrawal_request`
- [ ] Listen to `rider_location_updated` → update rider map markers
- [ ] Listen to `rider_status_updated` → update rider status indicators
- [ ] `admin_subscribe_rider` for detailed individual rider tracking
- [ ] All notifications auto-filtered by admin's assignedCity

---

## Important IDs Reference

| Context | Which ID to use | Field |
|---|---|---|
| Customer registration | User ID | `user.id` |
| Rider registration (both namespaces) | User ID | `user.id` |
| Vendor registration | Profile ID | `user.restaurant.id` or `user.shop.id` |
| Admin registration | User ID | `user.id` |
| Subscribe to order | Order ID | `order.id` (UUID) |
| Location updates | User ID | `user.id` (maps to `rider.userId`) |

---

*Last updated: February 2026*
