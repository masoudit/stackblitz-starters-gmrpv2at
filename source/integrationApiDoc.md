# Snapp Box API Reference

## Table of Contents

> For procedural integration guidance (authentication flow, mandatory calling sequences, order flows, implementation plan), see the [Client Integration Guide](./integrationGuide.md).

- [Overview](#overview)
- [Authentication](#authentication)
- [Orders v1](#orders-v1)
- [Orders v2](#orders-v2)
- [NXP v1](#nxp-v1)
- [Customers & Addresses](#customers--addresses)
- [Wallets & Insurance](#wallets--insurance)
- [Errors & Conventions](#errors--conventions)

## Overview

*API reference for external partners building clients against this gateway.*

### What is this?
Think of this service as the **front desk** of Snapp Box. Your systems never talk to internal services directly — they talk to this gateway. It checks your identity, validates your request, translates it, and forwards it to the right internal service.

### Base URL
Contact **SnappBox Support** to obtain the correct Base URL for the service.

### Endpoint Groups
| Group | Purpose |
|---|---|
| /v1/oauth2 | Get access token |
| /v1/orders | Create, track, update, cancel delivery orders |
| /v2/orders | Support both Same-day and Classic SnappBox services |
| /v1/nxp | Sender (NXP) orders: sizes, timelines, invoice, create, pay |
| /v1/customers | Favorite addresses, pickup verification |
| /v1/wallets | Balance lookup |
| /v1/insurances | Insurance options |
| /v1/pricing | Price quote before creating order |

### Golden Rules

- Every protected call needs `Authorization: Bearer <token>`

- Body format: `Content-Type: application/json`

- All times in **Asia/Tehran**, ISO-8601 with offset, e.g. `2025-08-06T19:00:00+03:30`

- Use a **unique refId** per business order — it is your idempotency key

- Fields marked **Required** must be present or the gateway returns 400

### Three Order Flows
This gateway exposes **three different order flows**. Pick the right one:

| Flow | Path | Use case |
|---|---|---|
| Standard orders | /v1/orders | Classic point-to-point delivery (bike, car...) |
| Standard orders v2 | /v2/orders | Both Same-day and Classic SnappBox services |
| NXP sender orders | /v1/nxp | Parcel shipping with timelines, invoices, payment |

Each flow has a **mandatory calling sequence** — see the [Client Integration Guide](./integrationGuide.md).

## Authentication

*For the complete authentication flow and token usage, see the [Client Integration Guide](./integrationGuide.md#authentication-flow). The endpoint reference is below.*

*One token to rule them all — get it first, reuse it everywhere.*

##### `POST /v1/oauth2/token`

  Exchange client credentials for an access token. **No auth header needed** for this one call.

  #### Request body
  | Field | Type | Required | Description |
|---|---|---|---|
| client_id | string | Required | Your issued client ID |
| client_secret | string | Required | Your issued client secret |

  #### Sample CURL
  
```bash
curl -X POST "https://<HOST>/v1/oauth2/token" \
  -H "Content-Type: application/json" \
  -d '{
    "client_id": "your-client-id",
    "client_secret": "your-client-secret"
  }'
```

  #### Response
  
```json
{
  "access_token": "<jwt>",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

## Orders v1

*The mandatory calling sequence is covered in the [Client Integration Guide](./integrationGuide.md#standard-orders-v1-flow). Endpoint reference follows.*

*Classic point-to-point delivery orders. Main base path: /v1/orders — this page also covers the related /v1/pricing and /v1/customers/verify-pickup-code endpoints. All endpoints require Authorization: Bearer <token>.*

:::

##### `GET /v1/orders/delivery-categories`

  List delivery categories available for a location. **All query parameters are mandatory.**

  #### Query params
  | Param | Type | Required | Example |
|---|---|---|---|
| latitude | number | Required | 35.757523 |
| longitude | number | Required | 51.409911 |

  #### Sample CURL
  
```bash
curl -X GET "https://<HOST>/v1/orders/delivery-categories?latitude=35.757523&longitude=51.409911" \
  -H "Authorization: Bearer <token>"
```

  #### Response (200)
  
```json
{
  "city": "tehran",
  "deliveryCategories": ["bike", "bike-without-box"]
}
```

##### `POST /v1/pricing`

  Get a price quote for a standard order **before** creating it. Mirrors the create-order shape: city, delivery category, terminals, packages. The `paymentType` field supports only these three values: `prepaid`, `cod`, `postpaid`.

  #### Sample CURL
  
```bash
curl -X POST "https://<HOST>/v1/pricing" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
  "city": "tehran",
  "deliveryCategory": "bike-without-box",
  "hasReturn": false,
  "packages": [
    {
      "dropoffReference": "2",
      "insuranceId": 10,
      "items": [
        {
          "name": "بادام نمکی مزمز 30 گرمی",
          "packageValue": 48000,
          "quantity": 1,
          "quantityMeasuringUnit": "عدد",
          "volume": 20,
          "weight": 10
        },
        {
          "name": "بادام نمکی مزمز 30 گرمی",
          "packageValue": 48000,
          "quantity": 1,
          "quantityMeasuringUnit": "عدد",
          "volume": 20,
          "weight": 10
        }
      ],
      "pickupReference": "1"
    },
    {
      "dropoffReference": "2",
      "insuranceId": 10,
      "items": [
        {
          "name": "بادام نمکی مزمز 30 گرمی",
          "packageValue": 48000,
          "quantity": 1,
          "quantityMeasuringUnit": "عدد",
          "volume": 20,
          "weight": 10
        },
        {
          "name": "بادام نمکی مزمز 30 گرمی",
          "packageValue": 48000,
          "quantity": 1,
          "quantityMeasuringUnit": "عدد",
          "volume": 20,
          "weight": 10
        }
      ],
      "pickupReference": "1"
    }
  ],
  "paymentType": "prepaid",
  "sequenceNumberDeliveryCollection": 1,
  "terminals": [
    {
      "address": "ونک، بزرگراه شهید حقانی",
      "comment": "Pick up fragile items.",
      "contactName": "رضا غلامی",
      "hasPickupCode": false,
      "latitude": "35.757523",
      "longitude": "51.409911",
      "phoneNumber": "09121112233",
      "reference": "1",
      "status": "PENDING",
      "type": "pickup"
    },
    {
      "address": "ونک، بزرگراه شهید حقانی",
      "comment": "Pick up fragile items.",
      "contactName": "رضا غلامی",
      "hasPickupCode": false,
      "latitude": "35.757523",
      "longitude": "51.409911",
      "phoneNumber": "09121112233",
      "reference": "2",
      "status": "PENDING",
      "type": "drop"
    }
  ]
}'
```

  #### Response (200)
  
```json
{
  "totalFare": 300000,
  "finalCustomerFare": 300000,
  "distance": 8.4,
  "eta": 45,
  "pricingId": "prc-123",
  "onDemandBiddingEnabled": false,
  "scheduledBiddingEnabled": false
}
```

:::note
There is also a v2 pricing endpoint at POST /v2/pricing with the same purpose for the v2 order flow.
:::

##### `POST /v1/orders`

  Create a new delivery order. This is the core endpoint.

  
:::note
POP (Proof of Pick) requires rollout. If you need this feature, please contact SnappBox Support.
:::

  #### Request DTO — OrderCreateRequest
  | Field | Type | Required | Description |
|---|---|---|---|
| refId | string | Required | Your unique business reference (idempotency key). e.g. box-545654 |
| terminals | array | Required | Pickup + dropoff stops. At least two element; each element must satisfy the Terminal required fields below. |
| deliveryCategory | string | Required | From `GET /v1/orders/delivery-categories`. e.g. bike-without-box |
| paymentType | string | Required | Only these three values are supported: prepaid, cod, postpaid |
| city | string | Optional | e.g. tehran |
| packages | array | Optional | Items being delivered |
| totalPackageSize | number | Optional | Total size |
| waitingTime | int | Optional | Minutes driver waits |
| voucherCode | string | Optional | Discount code |
| hasReturn | bool | Optional | Return trip needed |
| startTime / endTime | datetime | Optional | Delivery slot (Tehran time) |
| podEnabled | bool | Optional | Proof of delivery (verification code) |
| popEnabled | bool | Optional | Proof of pickup (pickup code) |

  #### Terminal object (inside terminals[])
  | Field | Type | Required | Description |
|---|---|---|---|
| latitude | string | Required | e.g. 35.757523 |
| longitude | string | Required | e.g. 51.409911 |
| type | string | Required | pickup or drop |
| reference | string | Required | Numeric string, defines stop order. e.g. "1" |
| phoneNumber | string | Required | Contact phone, e.g. 09121112233 |
| address | string | Optional | Full address text |
| contactName | string | Optional | Contact person |
| comment | string | Optional | Notes for driver |

  #### Sample CURL
  
```bash
curl -X POST "https://<HOST>/v1/orders" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
  "city": "tehran",
  "deliveryCategory": "bike-without-box",
  "hasReturn": false,
  "packages": [
    {
      "dropoffReference": "2",
      "insuranceId": 10,
      "items": [
        {
          "name": "بادام نمکی مزمز 30 گرمی",
          "packageValue": 48000,
          "quantity": 1,
          "quantityMeasuringUnit": "عدد",
          "volume": 20,
          "weight": 10
        },
        {
          "name": "بادام نمکی مزمز 30 گرمی",
          "packageValue": 48000,
          "quantity": 1,
          "quantityMeasuringUnit": "عدد",
          "volume": 20,
          "weight": 10
        }
      ],
      "pickupReference": "1"
    },
    {
      "dropoffReference": "2",
      "insuranceId": 10,
      "items": [
        {
          "name": "بادام نمکی مزمز 30 گرمی",
          "packageValue": 48000,
          "quantity": 1,
          "quantityMeasuringUnit": "عدد",
          "volume": 20,
          "weight": 10
        },
        {
          "name": "بادام نمکی مزمز 30 گرمی",
          "packageValue": 48000,
          "quantity": 1,
          "quantityMeasuringUnit": "عدد",
          "volume": 20,
          "weight": 10
        }
      ],
      "pickupReference": "1"
    }
  ],
  "paymentType": "prepaid",
  "sequenceNumberDeliveryCollection": 1,
  "terminals": [
    {
      "address": "ونک، بزرگراه شهید حقانی",
      "comment": "Pick up fragile items.",
      "contactName": "رضا غلامی",
      "hasPickupCode": false,
      "latitude": "35.757523",
      "longitude": "51.409911",
      "phoneNumber": "09121112233",
      "reference": "1",
      "status": "PENDING",
      "type": "pickup"
    },
    {
      "address": "ونک، بزرگراه شهید حقانی",
      "comment": "Pick up fragile items.",
      "contactName": "رضا غلامی",
      "hasPickupCode": false,
      "latitude": "35.757523",
      "longitude": "51.409911",
      "phoneNumber": "09121112233",
      "reference": "2",
      "status": "PENDING",
      "type": "drop"
    }
  ]
}'
```

  #### Response (201)
  
```json
{
  "orderId": "12345675",
  "refId": "box-545654",
  "status": "PENDING",
  "price": 300000,
  "createdTime": "2025-08-06 19:00:00",
  "terminalIds": [101, 102]
}
```

##### `GET /v1/orders/:orderId`

  Get full order detail: status, terminals, driver, tracking URL.

  
```bash
curl -X GET "https://<HOST>/v1/orders/12345675" \
  -H "Authorization: Bearer <token>"
```

  #### Response (200)
  
```json
{
  "orderId": "12345675",
  "refId": "box-545654",
  "status": "PICKED_UP",
  "deliveryCategory": "bike-without-box",
  "paymentType": "prepaid",
  "price": 300000,
  "canCancel": false,
  "trackingUrl": "https://track.snappbox.ir/12345675",
  "createdTime": "2025-08-06 19:00:00",
  "updatedTime": "2025-08-06 19:20:00",
  "terminals": [
    {
      "id": 101,
      "type": "pickup",
      "reference": "1",
      "latitude": "35.757523",
      "longitude": "51.409911",
      "address": "Vanak Sq.",
      "contactName": "Sara",
      "phoneNumber": "09121112233",
      "status": "PICKED_UP",
      "hasPickupCode": false
    },
    {
      "id": 102,
      "type": "drop",
      "reference": "2",
      "latitude": "35.7219",
      "longitude": "51.3347",
      "address": "Azadi Sq.",
      "contactName": "Reza",
      "phoneNumber": "09123334455",
      "status": "PENDING",
      "hasPickupCode": false
    }
  ],
  "items": [
    {"name": "Book", "quantity": 1, "weight": 10, "volume": 20, "packageValue": 48000, "quantityMeasuringUnit": "عدد"}
  ],
  "allotment": {
    "name": "Ali Driver",
    "phoneNumber": "09120001122",
    "vehiclePlateNumber": "12ب345-67",
    "imageUrl": "https://.../driver.jpg",
    "Status": "ON_THE_WAY"
  }
}
```

##### `GET /v1/orders/references/:refId`

  Same as above, but look up by **your** refId instead of platform orderId. Response shape is identical to `GET /v1/orders/:orderId`.

  
```bash
curl -X GET "https://<HOST>/v1/orders/references/box-545654" \
  -H "Authorization: Bearer <token>"
```

##### `GET /v1/orders/list`

  List your orders with pagination and optional status filter.

  | Param | Type | Required | Example |
|---|---|---|---|
| status | string | Optional | DELIVERED |
| pageNumber | int | Optional | 1 |
| pageSize | int | Optional | 10 |

  
```bash
curl -X GET "https://<HOST>/v1/orders/list?status=DELIVERED&pageNumber=1&pageSize=10" \
  -H "Authorization: Bearer <token>"
```

  #### Response (200)
  Array of order detail objects:

  
```json
[
  {
    "id": "12345675",
    "customerRefId": "box-545654",
    "status": "DELIVERED",
    "deliveryCategory": "bike-without-box",
    "paymentType": "prepaid",
    "price": 300000,
    "trackingUrl": "https://track.snappbox.ir/12345675",
    "createdAt": "2025-08-06 19:00:00",
    "terminals": [ ... ],
    "itemDetails": [ ... ]
  }
]
```

##### `GET /v1/orders/:orderId/current-location`

  Live driver location for an active order.

  
```bash
curl -X GET "https://<HOST>/v1/orders/12345675/current-location" \
  -H "Authorization: Bearer <token>"
```

  #### Response (200)
  
```json
{
  "latitude": 35.757523,
  "longitude": 51.409911
}
```

##### `GET /v1/orders/:orderId/events`

  Full event history of the order (created, accepted, picked up, delivered...).

  
```bash
curl -X GET "https://<HOST>/v1/orders/12345675/events" \
  -H "Authorization: Bearer <token>"
```

  #### Response (200)
  
```json
[
  {"type": "ORDER_CREATED", "actionAt": "2025-08-06T19:00:00+03:30"},
  {"type": "ACCEPTED", "actionAt": "2025-08-06T19:05:00+03:30"},
  {"type": "PICKED_UP", "actionAt": "2025-08-06T19:20:00+03:30"}
]
```

##### `PUT /v1/orders/:orderId`

  Update an existing order (payment type, waiting time, terminals, packages, time slot).

  
:::note
Caution: orders can be updated only after the ACCEPTED state and just before the PICKED_UP state.
:::

  
```bash
curl -X PUT "https://<HOST>/v1/orders/12345675" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "paymentType": "prepaid",
    "waitingTime": 10,
    "terminals": [ ... ],
    "packages": [ ... ]
  }'
```

  #### Response
  Returns `204` with no body on success.

##### `DELETE /v1/orders/:orderId`

  Cancel an order. Returns 204 on success.

  
```bash
curl -X DELETE "https://<HOST>/v1/orders/12345675" \
  -H "Authorization: Bearer <token>"
```

##### `POST /v1/orders/verify`

  Verify an order event (driver arrived at pickup, picked up, ready for pickup).

  #### Request DTO
  | Field | Type | Required | Description |
|---|---|---|---|
| orderId | string | Required | Platform order ID |
| customerRefId | string | Required | Your refId |
| verificationType | string | Required | ARRIVED_AT_PICKUP, PICKED_UP, or READY_FOR_PICKUP |

  
```bash
curl -X POST "https://<HOST>/v1/orders/verify" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "orderId": "12345675",
    "customerRefId": "box-545654",
    "verificationType": "PICKED_UP"
  }'
```

  #### Response
  Returns `204` with no body on success.

##### `POST /v1/orders/verify/picked-up/:refId`

  Shortcut: mark an order as verified-picked-up using your refId. Returns `204` with no body on success.

  
```bash
curl -X POST "https://<HOST>/v1/orders/verify/picked-up/box-545654" \
  -H "Authorization: Bearer <token>"
```

##### `POST /v1/customers/verify-pickup-code`

  Verify a pickup code handed to the driver.

  
:::note
POP (Proof of Pick) requires rollout. If you need this feature, please contact SnappBox Support.
:::

  #### Request DTO
  | Field | Type | Required | Description |
|---|---|---|---|
| pickupCode | string | Required | Pick up code |
| terminalId | string | Required | Terminal id of pickup terminal |
| customerRefId | string | Required | Your ref id |
| orderId | string | Required(Only if you don't send customerRefId) | Box Order id |
| referenceOrderId | string | Optional | Parent Order id of Batched orders |

  
```bash
curl -X POST "https://<HOST>/v1/customers/verify-pickup-code" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "orderId": "12345675",
    "terminalId": "101",
    "pickupCode": "12345678"
  }'
```

  #### Response
  Returns `200` on success.

##### `POST /v1/orders/number-masking`

  Apply phone number masking on an order terminal. Call this when one of your customers wants to call the biker — it hides their real phone numbers from each other. Returns 204 with no body.

  
:::note
This feature requires rollout. If you need this feature, please contact SnappBox Support.
:::

  #### Request DTO
  | Field | Type | Required | Description |
|---|---|---|---|
| orderId | string | Required | Platform order ID |
| terminalId | int | Required | Terminal ID |

  
```bash
curl -X POST "https://<HOST>/v1/orders/number-masking" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "orderId": "12345675",
    "terminalId": 101
  }'
```

### Order Status Values
| Status | Meaning |
|---|---|
| PENDING | Created, not yet progressed |
| ACCEPTED | Driver allocated |
| PICKED_UP | Package picked up |
| DELIVERED | Delivered |
| CANCELLED | Cancelled |

### Terminal Status Values
Each terminal (stop) on an order has its own status lifecycle, separate from the overall order status.

| Status | Meaning |
|---|---|
| PENDING | Stop not started |
| ARRIVED_AT_PICKUP | Driver at pickup stop |
| PICKED_UP | Picked up from this stop |
| ARRIVED_AT_DROP_OFF | Driver at drop-off stop |
| DELIVERED | Delivered at this stop |
| CANCELLED | Stop cancelled |
| FAILED | Stop failed |

## Orders v2

*The Same-day B2B calling sequence is covered in the [Client Integration Guide](./integrationGuide.md#same-day-b2b-order-flow-v2). Endpoint reference follows.*

*Base path: /v2/orders. These endpoints support both Same-day and Classic SnappBox services. All endpoints require Authorization: Bearer <token>.*

### Endpoints
| Method | Path | Purpose |
|---|---|---|
| `POST` | /v2/orders | Create an order (Classic or Same-day, chosen by deliveryCategory) |
| `GET` | /v2/orders/:orderId | Get order detail by platform order ID |
| `GET` | /v2/orders/references/:refId | Get order detail by your reference ID |

##### `POST /v2/orders`

  Create an order. Set `deliveryCategory` to `same-day` for Same-day service; any other category creates a Classic order. Quote first with `POST /v2/pricing`.

  #### Request DTO — OrderCreateRequest (v2)
  | Field | Type | Required | Description |
|---|---|---|---|
| refId | string | Optional (Required for Same-day) | Your unique business reference |
| terminals | array | Required | Pickup + dropoff stops. Each terminal requires latitude, longitude, type, reference |
| deliveryCategory | string | Required | e.g. same-day, bike-without-box |
| paymentType | string | Required | e.g. prepaid |
| city | string | Optional | e.g. tehran |
| timeSlot | object | Optional (Required for Same-day) | Time slot; must include id for Same-day |
| packages | array | Optional (Required for Same-day) | Items being delivered |
| features | object | Optional | Extra order features |
| totalPackageSize | number | Optional | Total size |
| waitingTime | int | Optional | Minutes driver waits |
| voucherCode | string | Optional | Discount code |
| hasReturn | bool | Optional | Return trip needed |
| startTime / endTime | datetime | Optional | Delivery slot (Tehran time) |
| podEnabled / popEnabled | bool | Optional | Proof of delivery / pickup |

  #### Sample CURL
  
```bash
curl -X POST "https://<HOST>/v2/orders" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "refId": "box-545654",
    "deliveryCategory": "bike-without-box",
    "paymentType": "prepaid",
    "city": "tehran",
    "terminals": [
      {
        "type": "pickup",
        "reference": "1",
        "latitude": "35.757523",
        "longitude": "51.409911",
        "address": "Vanak Sq.",
        "contactName": "Sara",
        "phoneNumber": "09121112233"
      },
      {
        "type": "drop",
        "reference": "2",
        "latitude": "35.7219",
        "longitude": "51.3347",
        "address": "Azadi Sq.",
        "contactName": "Reza",
        "phoneNumber": "09123334455"
      }
    ]
  }'
```

  #### Response (201)
  
```json
{
  "orderId": "12345675",
  "refId": "box-545654",
  "status": "PENDING",
  "price": 300000,
  "createdTime": "2025-08-06 19:00:00",
  "terminalIds": [101, 102],
  "packages": [
    {
      "verification_code": "123456",
      "pod_code": 1234,
      "tracking_code": "TRK123"
    }
  ]
}
```

##### `GET /v2/orders/:orderId`

  Get full order detail by platform order ID.

  
```bash
curl -X GET "https://<HOST>/v2/orders/12345675" \
  -H "Authorization: Bearer <token>"
```

  #### Response (200)
  
```json
{
  "orderId": "12345675",
  "refId": "box-545654",
  "status": "PICKED_UP",
  "deliveryCategory": "bike-without-box",
  "paymentType": "prepaid",
  "price": 300000,
  "canCancel": false,
  "trackingUrl": "https://track.snappbox.ir/12345675",
  "createdTime": "2025-08-06 19:00:00",
  "updatedTime": "2025-08-06 19:20:00",
  "terminals": [ ... ],
  "items": [ ... ],
  "allotment": {
    "name": "Ali Driver",
    "phoneNumber": "09120001122",
    "vehiclePlateNumber": "12ب345-67",
    "imageUrl": "https://.../driver.jpg",
    "status": "ON_THE_WAY"
  }
}
```

##### `GET /v2/orders/references/:refId`

  Get order detail by **your** reference ID. Response shape is identical to `GET /v2/orders/:orderId`.

  
```bash
curl -X GET "https://<HOST>/v2/orders/references/box-545654" \
  -H "Authorization: Bearer <token>"
```

#### POST /v1/nxp/sdd/timelines
Role in the Same-day flow: returns the time slots you can pick for pickup and delivery on the same day.

#### Request DTO — SddTimelineRequest
| Field | Type | Required | Description |
|---|---|---|---|
| parcels | array | Required | At least 1; each has a required size |
| sender_terminal_id | string | Required | Favorite Address ID, format fav_id:<id> |

```bash
curl -X POST "https://<HOST>/v1/nxp/sdd/timelines" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "sender_terminal_id": "fav_id:456",
    "parcels": [
      {"size": {"type": "slug", "value": "small"}}
    ]
  }'
```

#### Response (200)

```json
[
  {
    "id": "sdd-9001",
    "status": "available",
    "day": "2025-08-06",
    "start_time": "14:00",
    "end_time": "16:00",
    "reserved_count": 3
  }
]
```

## NXP v1

*The NXP pre-action requirements and mandatory calling sequence are covered in the [Client Integration Guide](./integrationGuide.md#nxp-sender-order-flow). Endpoint reference follows.*

*Sender/parcel shipping flow. Base path: /v1/nxp. All endpoints require Authorization: Bearer <token>.*

### The Size object (used everywhere)
| Field | Type | Required | Description |
|---|---|---|---|
| type | string | Required | category or dimension or slug |
| value | string | Required | category → size ID (e.g. "3"); dimension → "L,W,H" (e.g. "10,20,30"); slug → slug ID (e.g. "S3"); |

### Terminal IDs
Only the **pickup terminal** (`sender_terminal_id`) should have its ID filled. The accepted format is ONLY:

```
fav_id:<id>
```

Where `<id>` is the ID of a saved Favorite Address (see `POST /v1/customers/favorite-address`). Example: `fav_id:456`.

##### `GET /v1/nxp/sizes/:delivery_type`

  List parcel sizes for a delivery type (`standard` or `same-day`).

  
```bash
curl -X GET "https://<HOST>/v1/nxp/sizes/standard" \
  -H "Authorization: Bearer <token>" \
  -H "Accept-Language: fa_IR"
```

  #### Response (200)
  
```json
[
  {
    "id": 3,
    "key": "S1",
    "title": "سایز 1 پستی",
    "delivery_type": "standard",
    "description": "",
    "capacity": 5,
    "width": 20,
    "height": 10,
    "depth": 15
  }
]
```

##### `POST /v1/nxp/timelines`

  Search available time slots for standard delivery.

  #### Request DTO — SearchTimelinesRequest
  | Field | Type | Required | Description |
|---|---|---|---|
| parcels | array | Required | At least 1 parcel |

  #### Parcel object
  | Field | Type | Required | Description |
|---|---|---|---|
| size | Size | Required | See Size object above |
| sender_terminal_id | string | Required | e.g. fav_id:456 |
| receiver_terminal | object | Optional* | Full receiver address object |
| receiver_terminal_id | string | Optional* | e.g. fav_id:789 |

  
:::note
* Either receiver_terminal OR receiver_terminal_id must be provided — one of the two.
:::

  #### receiver_terminal object
  | Field | Type | Required | Description |
|---|---|---|---|
| address | string | Required | Full address |
| cellphone | string | Required | Receiver phone |
| city_id | int | Required | City ID, > 0 |
| name | string | Optional | Receiver name |
| latitude/longitude | number | Optional | Coordinates |
| postal_code | string | Optional | Postal code |

  #### Sample CURL
  
```bash
curl -X POST "https://<HOST>/v1/nxp/timelines" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "parcels": [
        {
            "sender_terminal_id": "fav_id:745",
            "size": {
                "title": "سایز 4 پستی",
                "type": "slug",
                "value": "S4"
            },
            "receiver_terminal": {
                "name": "سارا فرجی",
                "address": "خیابان ولیعصر، بالاتر از خیابان بهشتی، روبروی ایستگاه پله اول، کوچه دلبسته",
                "building_number": 12,
                "reference": "12345",
                "cellphone": "09228357845",
                "city_id": 1,
                "floor": 0,
                "door_number": 7,
                "postal_code": "3471957357",
                "latitude": null,
                "longitude": null
            }
        },
        {
            "sender_terminal_id": "fav_id:745",
            "size": {
                "title": "سایز 1 پستی",
                "type": "slug",
                "value": "S1"
            },
            "receiver_terminal": {
                "name": "علی حسینی",
                "address": "خیابان جردن، خیابان ناهید غربی",
                "building_number": 2,
                "reference": "98543",
                "cellphone": "09228357845",
                "city_id": 1,
                "floor": 0,
                "door_number": 3,
                "postal_code": "3471957357",
                "latitude": "35.8131420635902",
                "longitude": "51.398550441818"
            }
        }
    ]
}'
```

  #### Response (200)
  
```json
[
  {
    "id": "42585",
    "pickup_timeslot": {
      "id": "101",
      "hub_id": "5",
      "day": "2025-08-07",
      "start_time": "09:00",
      "end_time": "12:00",
      "city_id": 1
    },
    "delivery_timeslot": {
      "id": "202",
      "hub_id": "5",
      "day": "2025-08-08",
      "start_time": "14:00",
      "end_time": "18:00",
      "city_id": 1
    }
  }
]
```

##### `GET /v1/nxp/bundles`

  **Optional.** List your bundles. Use it when you already have an order in `ongoing` status — then you can choose the pickup date of your new order based on the existing `ON_GOING` bundle.

  #### Query params
  | Param | Type | Required | Example |
|---|---|---|---|
| bundle_status | string | Required | ongoing |
| sender_terminal_id | string | Required | Sender Terminal Favorite Address Id |
| page | int | Optional | 1 |
| page_size | int | Optional | 20 |

  
```bash
curl -X GET "https://<HOST>/v1/nxp/bundles?bundle_status=ongoing&page=1&page_size=20&sender_terminal_id=fav_id:745" \
  -H "Authorization: Bearer <token>"
```

  #### Response (200)
  
```json
[
  {
    "code": "BND-1001",
    "parcels_count": 4,
    "active_parcels_count": 2,
    "created_at": "2025-08-06T10:00:00Z",
    "updated_at": "2025-08-06T12:00:00Z",
    "pickup_timeslot": {
      "id": "101",
      "hub_id": "5",
      "day": "2025-08-07",
      "start_time": "09:00",
      "end_time": "12:00",
      "city_id": 1
    },
    "sender_terminal": {
      "id": "745",
      "title": "Office",
      "address": "Tehran, Vanak Sq.",
      "latitude": "35.757523",
      "longitude": "51.409911",
      "cellphone": "09121234567",
      "phone": "",
      "city_id": 1,
      "fav_id": 745,
      "created_at": "2025-08-01T10:00:00Z",
      "updated_at": "2025-08-01T10:00:00Z"
    }
  }
]
```

##### `POST /v1/nxp/orders/invoice`

  Get a price quote. Same body as create order. Returns pricing + `token` (used as `invoice_token` when creating the order).

  
```bash
curl -X POST "https://<HOST>/v1/nxp/orders/invoice" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "reference": "order-8c7c8b8cc04c41c092627fec65c995e9",
    "parcels": [
        {
            "sender_terminal_id": "fav_id:745",
            "receiver_terminal": {
                "name": "سارا فرجی",
                "address": "خیابان ولیعصر، بالاتر از خیابان بهشتی، روبروی ایستگاه پله اول، کوچه دلبسته",
                "building_number": 12,
                "cellphone": "09228357845",
                "city_id": 1,
                "floor": 0,
                "door_number": 7,
                "postal_code": "3471957357",
                "latitude": null,
                "longitude": null
            },
            "size": {
                "title": "سایز 4 پستی",
                "type": "slug",
                "value": "S4"
            },
            "weight": {
                "type": "value",
                "value": "500"
            },
            "reference": "12345",
            "timeline_id": "42585",
            "pickup_type": "snapp-box",
            "value": 100000000,
            "breakable": false
        },
        {
            "sender_terminal_id": "fav_id:745",
            "receiver_terminal": {
                "name": "علی حسینی",
                "address": "خیابان جردن، خیابان ناهید غربی",
                "building_number": 2,
                "cellphone": "09228357845",
                "city_id": 1,
                "floor": 0,
                "door_number": 3,
                "postal_code": "3471957357",
                "latitude": "35.8131420635902",
                "longitude": "51.398550441818"
            },
            "size": {
                "title": "سایز 1 پستی",
                "type": "slug",
                "value": "S1"
            },
            "weight": {
                "type": "value",
                "value": "350"
            },
            "reference": "98543",
            "timeline_id": "42585",
            "pickup_type": "snapp-box",
            "value": 50000000,
            "breakable": false
        }
    ]
}'
```

  #### Response (200)
  
```json
{
  "total": 850000,
  "base_price": 700000,
  "first_mile": 50000,
  "mid_mile": 0,
  "last_mile": 50000,
  "discount": 0,
  "insurance": 50000,
  "tax": 0,
  "reference": "order-8c7c8b8cc04c41c092627fec65c995e9",
  "token": "0oRDoQEnoQRUK/bj9nfSp9wk3..."
}
```

##### `POST /v1/nxp/orders`

  Create the order.

  #### Request DTO — CreateOrderReq
  | Field | Type | Required | Description |
|---|---|---|---|
| invoice_token | string | Required | Token from the `POST /v1/nxp/orders/invoice` response |
| parcels | array | Required | At least 1 parcel |
| reference | string | Optional | Your reference, 1–50 chars |

  #### CreateParcelReq object
  | Field | Type | Required | Description |
|---|---|---|---|
| timeline_id | string | Required | From `POST /v1/nxp/timelines` |
| sender_terminal_id | string | Required | e.g. fav_id:456 |
| size | Size | Required | Size object |
| receiver_terminal / receiver_terminal_id | object/string | One required | Receiver info |
| pickup_type | string | Optional | snapp-box (default) or drop_off |
| value | int | Optional | Declared value, >= 0 |
| breakable / packaging | bool | Optional | Flags, default false |

  #### Sample CURL
  
```bash
curl -X POST "https://<HOST>/v1/nxp/orders" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -H "X-Idempotency-Key: <unique-uuid>" \
  -d '{
    "invoice_token": "0oRDoQEnoQRUK/bj9nfSp9wk3BewVjSy6NumxOdZBHGqYTEaAAr8gGEygqZhMVQXLXW2Jc6LLSPXE9ujrMyg4pyd8GEyGgAGQZBhMxoABkGQYTQAYTWyYTFlMTIzNDVhMmJTNGEzAmE0omExTP////ECfsqfOsLS+2EyTP////ECtpKRvP3EBmE1AWE2omExTP////ECfku92YJgOmEyTP////ICEmkPTrekRmE3AWE49WE59WIxMPViMTEaBfXhAGIxMvViMTP0YjE09GIxNaJhMRpqoBvoYTIaaqA4CGIxNqJhMRpqoW1oYTIaaqGJiGIxN6JhMUn////4AtUh6AxhMkr////4AgEybVJYYjE4omExSf////gC1SHoDGEySv////gCATJtUlhhNrgeYTEZYahhMgBhMxkD6GE0GSLEYTUZYahhNhoABd/oYTcAYTgAYTkZnEBiMTAZ1thiMTEAYjEyAGIxMwBiMTQAYjE1AGIxNhoAAYagYjE3AGIxOABiMTkAYjIwGgADDUBiMjEZdTBiMjIZOphiMjMAYjI0GTqYYjI1+0BZAAAAAAAAYjI2+wAAAAAAAAAAYjI3+wAAAAAAAAAAYjI4b0JPWC0yNjY5NCwyNjY4MGIyOXgtQk9YLTI2NjkyLDI2Njg4LDI2Njg3LDI2Njg2LDI2Njg1LDI2Njg0LDI2NjgxYjMwYKZhMVQXLXW2lYT29DTCkoJldPlnl5jirmEyGgAEuvBhMxoABLrwYTQAYTWyYTFlOTg1NDNhMmJTMWEzAmE0omExTP////ECfsqfOsLS+2EyTP////ECtpKRvP3EBmE1AWE2omExTP////MCAUW3+Bwi/mEyS/////QCLr8oV09aYTcBYTj0YTn1YjEw9WIxMRoF9eEAYjEy9WIxM/RiMTT0YjE1omExGmqgG+hhMhpqoDgIYjE2omExGmqhbWhhMhpqoYmIYjE3omExSf////gC1SHoDGEySv////gCATJtUlhiMTiiYTFJ////+ALVIegMYTJK////+AIBMm1SWGE2uB5hMRlhqGEyAGEzGQPoYTQZIsRhNRlhqGE2GgAEWUhhNwBhOABhORmcQGIxMBnW2GIxMQBiMTIAYjEzAGIxNABiMTUAYjE2AGIxNwBiMTgAYjE5AGIyMBoAAw1AYjIxGXUwYjIyGTqYYjIzAGIyNBk6mGIyNftAWQAAAAAAAGIyNvsAAAAAAAAAAGIyN/sAAAAAAAAAAGIyOG9CT1gtMjY2OTQsMjY2ODBiMjl4RUJPWC0yNjY5MSwyNjY5MCwyNjY4OSwyNjY5MiwyNjY4OCwyNjY4NywyNjY4NiwyNjY4NSwyNjY4NCwyNjY4MSwyNjY3N2IzMGBhMxpqn8O2YTQZ6mBhNRoACvyAYTYAYTcAYThYIMLNdK7p3wd/maz2ZRuHPaDE2DmVXi3v+lXJ+49Y/od3YTlYIEM7FWy6/wL5NWFOmtkbxj8Aj5ivs+6EvrH5tBnVRpKjYjEwo2ExomVWYWxpZPRmU3RyaW5nYGEy+z+5mZmZmZmaYTMZebdYQCXMrOLUPEu/awopiBkuLLQ5lfosA+dfc2+o0enJkdY8N308B0Sii8y1PI8ljidGtFuIKQAwx6lMssuleVZo/gY=",
    "parcels": [
        {
            "sender_terminal_id": "fav_id:745",
            "receiver_terminal": {
                "name": "سارا فرجی",
                "address": "خیابان ولیعصر، بالاتر از خیابان بهشتی، روبروی ایستگاه پله اول، کوچه دلبسته",
                "building_number": 12,
                "cellphone": "09228357845",
                "city_id": 1,
                "floor": 0,
                "door_number": 7,
                "postal_code": "3471957357",
                "latitude": null,
                "longitude": null
            },
            "size": {
                "title": "سایز 4 پستی",
                "type": "slug",
                "value": "S4"
            },
            "weight": {
                "type": "value",
                "value": "500"
            },
            "reference": "12345",
            "timeline_id": "42585",
            "pickup_type": "snapp-box",
            "value": 100000000,
            "breakable": false
        },
        {
            "sender_terminal_id": "fav_id:745",
            "receiver_terminal": {
                "name": "علی حسینی",
                "address": "خیابان جردن، خیابان ناهید غربی",
                "building_number": 2,
                "cellphone": "09228357845",
                "city_id": 1,
                "floor": 0,
                "door_number": 3,
                "postal_code": "3471957357",
                "latitude": "35.8131420635902",
                "longitude": "51.398550441818"
            },
            "size": {
                "title": "سایز 1 پستی",
                "type": "slug",
                "value": "S1"
            },
            "weight": {
                "type": "value",
                "value": "350"
            },
            "reference": "98543",
            "timeline_id": "42585",
            "pickup_type": "snapp-box",
            "value": 50000000,
            "breakable": false
        }
    ]
}'
```

  #### Response (201)
  
```json
{
  "id": "12345675",
  "status": "created",
  "reference": "my-order-001",
  "parcels_count": 2,
  "amount": 850000,
  "created_at": "2025-08-06T19:00:00Z",
  "updated_at": "2025-08-06T19:00:00Z",
  "tags": {
    "pickup_timeslots": [{"id": "101", "day": "2025-08-07", "start_time": "09:00", "end_time": "12:00"}],
    "delivery_timeslots": [{"id": "202", "day": "2025-08-08", "start_time": "14:00", "end_time": "18:00"}],
    "sender_terminals": [{"id": "745", "title": "Office"}],
    "pickup_types": ["snapp-box"]
  },
  "timelines": [
    {"id": "42585", "status": "active", "order_cutoff": "2025-08-06T23:00:00Z"}
  ]
}
```

  
:::note
Send a unique X-Idempotency-Key header so retries never create duplicate orders.
:::

##### `POST /v1/nxp/orders/:orderId/transactions`

  Pay the order from your Snapp Box wallet. No request body needed.

  
```bash
curl -X POST "https://<HOST>/v1/nxp/orders/12345675/transactions" \
  -H "Authorization: Bearer <token>"
```

  #### Response (200)
  
```json
{
  "url": "",
  "paid": true
}
```

##### `GET /v1/nxp/orders`

  Search your sender orders with filters and pagination. Response shape is the same as `POST /v1/nxp/orders` (array of order objects).

  
```bash
curl -X GET "https://<HOST>/v1/nxp/orders?statuses=created,paid&page=1&page_size=20" \
  -H "Authorization: Bearer <token>"
```

  #### Response (200)
  
```json
[
  {
    "id": "12345675",
    "status": "paid",
    "reference": "my-order-001",
    "parcels_count": 2,
    "amount": 850000,
    "created_at": "2025-08-06T19:00:00Z",
    "updated_at": "2025-08-06T19:05:00Z",
    "tags": { ... },
    "timelines": [ ... ]
  }
]
```

##### `GET /v1/nxp/parcels`

  Search parcels with many filters (status, tracking code, terminal IDs...).

  
```bash
curl -X GET "https://<HOST>/v1/nxp/parcels?tracking_code=TRK123&page=1" \
  -H "Authorization: Bearer <token>"
```

  #### Response (200)
  
```json
[
  {
    "id": "555",
    "order_id": "12345675",
    "timeline_id": "42585",
    "tracking_code": "TRK123",
    "status": "created",
    "reference": "98543",
    "verification_code": 123456,
    "value": 50000000,
    "breakable": false,
    "created_at": "2025-08-06T19:00:00Z",
    "updated_at": "2025-08-06T19:00:00Z",
    "weight": {"value": 350, "category": ""},
    "size": {"value": "S1", "category": 0},
    "sender_terminal": { ... },
    "receiver_terminal": { ... },
    "pickup_timeslot": { ... },
    "delivery_timeslot": { ... }
  }
]
```

##### `GET /v1/nxp/parcels/labels`

  Download parcel barcode labels as an HTML page for printing. Response is an HTML page (`Content-Type: text/html`), not JSON.

  
```bash
curl -X GET "https://<HOST>/v1/nxp/parcels/labels?order_id=12345675" \
  -H "Authorization: Bearer <token>"
```

##### `DELETE /v1/nxp/parcels/:id`

  Cancel a sender parcel. Returns `204` with no body on success.

  
```bash
curl -X DELETE "https://<HOST>/v1/nxp/parcels/555" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"reason_id": 1}'
```

##### `GET /v1/nxp/transactions`

  List payment transactions with filters.

  
```bash
curl -X GET "https://<HOST>/v1/nxp/transactions?order_id=12345675" \
  -H "Authorization: Bearer <token>"
```

  #### Response (200)
  
```json
[
  {
    "transaction_id": "txn-9001",
    "status": "paid",
    "type": "order_payment",
    "gateway": "wallet",
    "paid_amount": 850000,
    "remaining": 0,
    "total_amount": 850000,
    "finalized_at": "2025-08-06T19:05:00Z"
  }
]
```

## Customers & Addresses

*Favorite addresses. Base path: /v1/customers.*

### Favorite Addresses
Saved addresses your users can reuse when creating orders. Think of it as an address book.

##### `POST /v1/customers/favorite-address`

  Save a new favorite address.

  #### Request DTO — FavoriteAddress
  | Field | Type | Required | Description |
|---|---|---|---|
| name | string | Required | Label, e.g. "Office". Must not be blank (whitespace only rejected) |
| contactName | string | Required | Contact person |
| contactPhoneNumber | string | Required | Iranian mobile, e.g. 09121234567 |
| latitude | string | Required | e.g. 35.757523 |
| longitude | string | Required | e.g. 51.409911 |
| address | string | Required | Full address text |
| plate / unit / comment | string | Optional | Extra location details |
| defaultAddress | bool | Optional | Set as default |

  #### Sample CURL
  
```bash
curl -X POST "https://<HOST>/v1/customers/favorite-address" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Office",
    "contactName": "Sara Ahmadi",
    "contactPhoneNumber": "09121234567",
    "latitude": "35.757523",
    "longitude": "51.409911",
    "address": "Tehran, Vanak Sq.",
    "plate": "12",
    "unit": "4",
    "defaultAddress": true
  }'
```

  #### Response (201)
  
```json
{
  "id": "456",
  "name": "Office",
  "contactName": "Sara Ahmadi",
  "contactPhoneNumber": "09121234567",
  "latitude": "35.757523",
  "longitude": "51.409911",
  "address": "Tehran, Vanak Sq.",
  "plate": "12",
  "unit": "4",
  "comment": "",
  "defaultAddress": true,
  "usageCount": 0
}
```

##### `GET /v1/customers/favorite-address`

  List saved favorite addresses.

  
```bash
curl -X GET "https://<HOST>/v1/customers/favorite-address" \
  -H "Authorization: Bearer <token>"
```

  #### Response (200)
  
```json
[
  {
    "id": "456",
    "name": "Office",
    "contactName": "Sara Ahmadi",
    "contactPhoneNumber": "09121234567",
    "latitude": "35.757523",
    "longitude": "51.409911",
    "address": "Tehran, Vanak Sq.",
    "plate": "12",
    "unit": "4",
    "comment": "",
    "defaultAddress": true,
    "usageCount": 3
  }
]
```

## Wallets & Insurance

*Helper endpoints for balance and insurance options.*

### Wallet

##### `GET /v1/wallets`

  Get your current wallet balance. NXP orders are paid from this wallet — check balance before the payment step.

  
```bash
curl -X GET "https://<HOST>/v1/wallets" \
  -H "Authorization: Bearer <token>"
```

  #### Response (200)
  
```json
{
  "balance": 5000000,
  "currency": "IRR"
}
```

### Insurance

##### `GET /v1/insurances`

  List available insurance options (package value tiers and their cost). Use the insurance `id` in the `insuranceId` field of order packages.

  
```bash
curl -X GET "https://<HOST>/v1/insurances?city=sampleCity&deliveryCategory=sampleCategory" \
  -H "Authorization: Bearer <token>"
```

  #### Response (200)
  
```json
{
  "city": "tehran",
  "deliveryCategory": "bike-without-box",
  "packageValues": [
    {
      "id": 10,
      "title": "Up to 500,000 IRR",
      "value": 500000,
      "insuranceAmount": 5000
    }
  ]
}
```

## Errors & Conventions

*How failures look, and the rules that apply everywhere.*

### Error Response Shape
Every failed call returns the same JSON shape:

```json
{
  "key": "invalid.request.payload",
  "message": "Invalid request payload",
  "statusCode": "400"
}
```

### Common HTTP Statuses
| Status | Meaning | What to do |
|---|---|---|
| 400 | Bad request — missing/invalid fields | Check the key field (see Request Validation above); fix the request body. Required fields are marked in each endpoint's DTO table. |
| 401 | Missing or expired token | Get a fresh token from /v1/oauth2/token and retry. |
| 404 | Resource not found | Check the ID in the path. |
| 500 | Server/upstream failure | Retry with backoff; contact support if persistent. |

### Conventions
#### Timezone
All date/time fields use **Asia/Tehran** (UTC+3:30), ISO-8601 with offset:

```
2025-08-06T19:00:00+03:30
```

#### Identifiers
| Field | Meaning |
|---|---|
| orderId | Snapp platform order ID — returned by create; use for get/cancel/events |
| refId | Your business reference — use for idempotency and lookup by reference |

:::note
Always send a unique refId per business order. Retrying a create with the same refId is safe.
:::

#### Content type

```
Content-Type: application/json
Accept: application/json
```

#### Phone numbers
The only acceptable phone number format is an 11-digit number, such as `09124567891`.

