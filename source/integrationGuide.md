# Snapp Box Client Integration Guide

*Procedural guide for external partners integrating with the Snapp Box gateway. For endpoint definitions, request/response DTOs, and field-level details, see the [API Reference](./integrationApiDoc.md).*

## Table of Contents

- [Before You Start](#before-you-start)
- [Authentication Flow](#authentication-flow)
- [Choosing an Order Flow](#choosing-an-order-flow)
- [Standard Orders (v1) Flow](#standard-orders-v1-flow)
- [Same-day B2B Order Flow (v2)](#same-day-b2b-order-flow-v2)
- [NXP Sender Order Flow](#nxp-sender-order-flow)
- [Integration Checklist](#integration-checklist)
- [Implementation Plan](#implementation-plan)

---

## Before You Start

Complete these preconditions before writing any integration code:

1. **Obtain credentials** — contact SnappBox Support to receive your `client_id` and `client_secret`.
2. **Obtain the Base URL** — contact SnappBox Support for the correct Base URL for the service.
3. **Choose your order flow** — the gateway exposes three different order flows (see [Choosing an Order Flow](#choosing-an-order-flow)). Decide which one(s) your client needs.
4. **Request feature rollouts if needed** — the following features require rollout by SnappBox. Contact SnappBox Support if you need them:
   - POP (Proof of Pick / pickup codes)
   - Phone number masking
5. **For NXP sender orders only** — pre-action requirement: save the sender's pickup address as a Favorite Address via `POST /v1/customers/favorite-address`. Every NXP endpoint identifies the sender terminal as `fav_id:<id>` of that saved address.

---

## Authentication Flow

### How it works

One token to rule them all — get it first, reuse it everywhere.

1. Call `POST /v1/oauth2/token` with your `client_id` and `client_secret`. This is the **only** call that needs no auth header.
2. The response contains an `access_token` (a JWT), `token_type: Bearer`, and `expires_in` (seconds, e.g. `3600`).

### Using the token

- Send the token on every protected call as:

  ```
  Authorization: Bearer <token>
  ```

- Reuse the token until it expires.

### Handling expiry

- When a call returns `401`, the token is missing or expired: request a fresh token from `POST /v1/oauth2/token` and retry the call.

See the [API Reference — Authentication](./integrationApiDoc.md#authentication) for the endpoint definition.

---

## Choosing an Order Flow

This gateway exposes **three different order flows**. Pick the right one:

| Flow | Base path | Use case |
|---|---|---|
| Standard orders | /v1/orders | Classic point-to-point delivery (bike, car...) |
| Standard orders v2 | /v2/orders | Both Same-day and Classic SnappBox services |
| NXP sender orders | /v1/nxp | Parcel shipping with timelines, invoices, payment |

Each flow has a **mandatory calling sequence**, described below.

---

## Standard Orders (v1) Flow

*Classic point-to-point delivery. All endpoint details: [API Reference — Orders v1](./integrationApiDoc.md#orders-v1).*

### Mandatory Calling Sequence

1. **List delivery categories** — `GET /v1/orders/delivery-categories` with the pickup `latitude`/`longitude`. All query parameters are mandatory. Use one of the returned categories as `deliveryCategory` in the next steps.
2. **Get a price quote** — `POST /v1/pricing` with the same shape you will use to create the order (city, delivery category, terminals, packages).
3. **Create the order** — `POST /v1/orders` with a unique `refId` per business order (your idempotency key).
4. **Track the order** — after creation, use:
   - `GET /v1/orders/:orderId` (or `GET /v1/orders/references/:refId`) for full order detail, status, and `trackingUrl`
   - `GET /v1/orders/:orderId/current-location` for the live driver location
   - `GET /v1/orders/:orderId/events` for the full event history

### Optional operations during the order lifecycle

- **Update** — `PUT /v1/orders/:orderId`. Caution: orders can be updated only after the `ACCEPTED` state and just before the `PICKED_UP` state.
- **Cancel** — `DELETE /v1/orders/:orderId`.
- **Verify order events** — `POST /v1/orders/verify` (driver arrived at pickup, picked up, ready for pickup), or the shortcut `POST /v1/orders/verify/picked-up/:refId`.
- **Verify pickup code (POP)** — `POST /v1/customers/verify-pickup-code`. Requires rollout — contact SnappBox Support.
- **Number masking** — `POST /v1/orders/number-masking` when a customer wants to call the biker without revealing real phone numbers. Requires rollout — contact SnappBox Support.

---

## Same-day B2B Order Flow (v2)

*Base path: /v2/orders. Supports both Same-day and Classic SnappBox services. All endpoint details: [API Reference — Orders v2](./integrationApiDoc.md#orders-v2).*

### How it works

The v2 create endpoint serves both services: set `deliveryCategory` to `same-day` for Same-day service; any other category creates a Classic order. For Same-day orders, `refId`, `timeSlot` (including the time slot `id`), and `packages` are required.

### Mandatory Calling Sequence (Same-day)

1. **Get same-day time slots** — `POST /v1/nxp/sdd/timelines` with the sender terminal (`fav_id:<id>` of a saved Favorite Address) and the parcel sizes. It returns the time slots you can pick for pickup and delivery on the same day — keep the chosen slot's `id`.
2. **Get a price quote** — `POST /v2/pricing` (same purpose as v1 pricing, for the v2 order flow).
3. **Create the order** — `POST /v2/orders` with `deliveryCategory: same-day`, your unique `refId`, and the chosen `timeSlot` (must include the slot `id`).
4. **Track the order** — `GET /v2/orders/:orderId` or `GET /v2/orders/references/:refId`.

---

## NXP Sender Order Flow

*Sender/parcel shipping flow. Base path: /v1/nxp. All endpoint details: [API Reference — NXP v1](./integrationApiDoc.md#nxp-v1).*

### Pre-action (required once per sender address)

Save the sender's pickup address as a **Favorite Address** via `POST /v1/customers/favorite-address`. NXP endpoints identify the sender terminal **only** in the format `fav_id:<id>`, where `<id>` is the ID of that saved address (e.g. `fav_id:456`). Only the pickup terminal (`sender_terminal_id`) should have its ID filled.

### Mandatory Calling Sequence

1. **List parcel sizes** — `GET /v1/nxp/sizes/:delivery_type` (`standard` or `same-day`) to pick a size for each parcel.
2. **Search timelines** — `POST /v1/nxp/timelines` with the parcels (each with a size and sender terminal) to get available pickup/delivery time slots. Keep the `timeline_id`.
   - *Optional:* if you already have an order in `ongoing` status, call `GET /v1/nxp/bundles` and choose the pickup date of your new order based on the existing `ON_GOING` bundle.
3. **Get an invoice** — `POST /v1/nxp/orders/invoice` with the same body as create order. The response returns pricing plus a `token`.
4. **Create the order** — `POST /v1/nxp/orders` with `invoice_token` from the invoice response and the parcels (each referencing the `timeline_id` from step 2). Send a unique `X-Idempotency-Key` header so retries never create duplicate orders.
5. **Pay the order** — `POST /v1/nxp/orders/:orderId/transactions`. Payment is taken from your Snapp Box wallet — check the balance with `GET /v1/wallets` before this step.

### After creation

- `GET /v1/nxp/orders` — search your sender orders.
- `GET /v1/nxp/parcels` — search parcels (by status, tracking code, terminal IDs...).
- `GET /v1/nxp/parcels/labels` — download parcel barcode labels as a printable HTML page.
- `DELETE /v1/nxp/parcels/:id` — cancel a parcel.
- `GET /v1/nxp/transactions` — list payment transactions.

---

## Integration Checklist

### General (all flows)

- [ ] `client_id` / `client_secret` and Base URL obtained from SnappBox Support
- [ ] Token flow implemented: obtain, reuse, refresh on `401`
- [ ] Every protected call sends `Authorization: Bearer <token>`
- [ ] All request bodies sent as `Content-Type: application/json`
- [ ] All date/times handled in **Asia/Tehran**, ISO-8601 with offset (e.g. `2025-08-06T19:00:00+03:30`)
- [ ] Phone numbers sent as 11-digit numbers (e.g. `09124567891`)
- [ ] Unique `refId` generated per business order (idempotency key)
- [ ] All fields marked **Required** in the API Reference are always sent
- [ ] Error handling parses the standard error shape (`key`, `message`, `statusCode`)

### Standard orders (v1)

- [ ] Sequence implemented: delivery-categories → pricing → create → track
- [ ] Update only attempted between `ACCEPTED` and `PICKED_UP` states
- [ ] (Optional) POP / number masking rollout requested

### Same-day B2B (v2)

- [ ] Sender address saved as Favorite Address; `fav_id:<id>` available
- [ ] Sequence implemented: sdd/timelines → v2 pricing → v2 create (with `refId`, `timeSlot.id`, `packages`) → track

### NXP sender orders

- [ ] Sender address saved as Favorite Address; `fav_id:<id>` used for `sender_terminal_id`
- [ ] Sequence implemented: sizes → timelines → invoice → create → pay
- [ ] Unique `X-Idempotency-Key` sent on order creation
- [ ] Wallet balance checked (`GET /v1/wallets`) before the payment step

## Implementation Plan

*Step-by-step plan for your development team to build a client-side integration with the Snapp Box B2B API. Language-agnostic — implement in any modern programming language.*

### 1. Introduction
This integration lets your system create, track, update, and cancel Snapp Box delivery orders through the B2B gateway. The gateway is the front desk of Snapp Box: your system never talks to internal services directly.

Supported services:

- **Classic** — point-to-point delivery (bike, car...) via `/v1/orders` or `/v2/orders`.

- **Same-day** — same-day delivery via `/v2/orders` with `deliveryCategory: "same-day"`.

- **NXP sender orders** — parcel shipping with timelines, invoices, and wallet payment via `/v1/nxp`.

Overall order lifecycle: quote price → create order → order moves through statuses (`PENDING` → `ACCEPTED` → `PICKED_UP` → `DELIVERED`, or `CANCELLED`) → track via detail/events endpoints.

### 2. Integration Architecture
Recommended client-side layers (any language/framework):

```
Client Application
→ API Client / HTTP Layer      (raw HTTP calls, headers, timeouts)
→ DTO / Serialization Layer    (request/response models, JSON serialize/deserialize)
→ Business Service             (order flows, sequence orchestration)
→ Validation                   (client-side checks before sending)
→ Error Handling               (map HTTP/API errors to your domain errors)
→ Logging / Monitoring         (correlation, latency, failures)
```

| Layer | Responsibility |
|---|---|
| API Client / HTTP | Send requests, attach Authorization: Bearer header, enforce timeouts, return raw responses. |
| DTO / Serialization | Define all request/response models (section 5). Serialize requests to JSON; deserialize responses. Tolerate unknown/extra fields in responses. |
| Business Service | Implement the order flows (section 6) in the documented call order. Store identifiers between steps. |
| Validation | Check required fields, formats, and enums before sending (section 8). Fail fast locally instead of getting a 400. |
| Error Handling | Parse the error response shape, decide retry vs. fail (section 10). |
| Logging / Monitoring | Correlation IDs, order IDs, status codes, latency (section 15). |

### 3. Configuration
Make these configurable (environment variables, config files, or a config service):

| Setting | Notes |
|---|---|
| Base URL | Obtain from SnappBox Support. Do not hardcode a URL from documentation. |
| client_id / client_secret | Issued credentials for POST /v1/oauth2/token. Store securely (secret manager), never in source code or logs. |
| HTTP timeout | Recommended client-side setting. No value is documented — see Documentation Gaps. |
| Retry configuration | Recommended client-side setting. Not documented — see section 11 and Documentation Gaps. |
| Environment | Separate config per environment (staging/production) if SnappBox provides multiple Base URLs. |

### 4. API Client Layer
Rules that apply to every endpoint:

- All endpoints except `POST /v1/oauth2/token` require header `Authorization: Bearer <access_token>`.

- Requests with a body need `Content-Type: application/json`.

- Date/time fields use **Asia/Tehran**, ISO-8601 with offset (e.g. `2025-08-06T19:00:00+03:30`).

- Fields marked **Required** must be present or the gateway returns 400.

Endpoint-by-endpoint details (method, path, params, bodies, statuses) are in the endpoint pages: [Orders v1](#orders), [Orders v2](#orders-v2), [NXP](#nxp), [Customers](#customers), [Wallets & Insurance](#wallets), [Authentication](#auth). The checklist in section 7 lists every endpoint you must implement.

Implementation requirements per endpoint:

- Build URLs from configurable Base URL + documented path; substitute path parameters (`:orderId`, `:refId`, `:delivery_type`, `:id`).

- Serialize query parameters exactly as documented; note that `GET /v1/orders/delivery-categories` requires **all** query parameters, and `GET /v1/nxp/bundles` requires `bundle_status` and `sender_terminal_id`.

- Expect documented success statuses: 200 for reads/actions, 201 for order/address creation, 204 (no body) for update/cancel/verify/number-masking.

### 5. DTO / Data Model Implementation
Implement every model below. Field names are JSON keys exactly as the API expects/returns them. "Nullable" means the field may be null or absent — your deserializer must tolerate both.

#### 5.1 Authentication
<h4>TokenRequest (request)</h4>
| Field | Type | Required | Description |
|---|---|---|---|
| client_id | string | Required | Issued client ID |
| client_secret | string | Required | Issued client secret |

<h4>TokenResponse (response)</h4>
| Field | Type | Nullable | Description |
|---|---|---|---|
| access_token | string | No | JWT to send as Bearer token |
| token_type | string | No | Bearer |
| expires_in | number | No | Token lifetime in seconds; when you get a 401, request a new token and retry |

#### 5.2 Orders v1 (Classic)
<h4>DeliveryCategoriesResponse (response)</h4>
| Field | Type | Nullable | Description |
|---|---|---|---|
| city | string | No | City the categories apply to |
| deliveryCategories | string[] | No | Valid deliveryCategory values, e.g. bike, bike-without-box |

<h4>PricingResponse (response)</h4>
| Field | Type | Nullable | Description |
|---|---|---|---|
| totalFare | int | No | Total price |
| finalCustomerFare | int | No | Price charged to customer |
| distance | number | No | Route distance |
| eta | number | No | Estimated minutes |
| pricingId | string | Yes | Quote identifier |
| onDemandBiddingEnabled / scheduledBiddingEnabled | bool | No | Bidding flags |

:::note
PricingResponse contains additional optional pricing fields (subsidy, surge, voucher info...). Treat all unlisted fields as optional and ignore unknown fields.
:::

<h4>OrderCreateRequest v1 (request)</h4>
| Field | Type | Required | Description |
|---|---|---|---|
| refId | string | Required | Your unique business reference (idempotency key) |
| terminals | Terminal[] | Required | At least one pickup and one drop |
| deliveryCategory | string | Required | From delivery-categories step |
| paymentType | string | Required | e.g. prepaid |
| city | string | Required | e.g. tehran |
| packages | array | Optional | Items being delivered |
| totalPackageSize | number | Optional | Total size |
| waitingTime | int | Optional | Minutes driver waits |
| voucherCode | string | Optional | Discount code |
| hasReturn | bool | Optional | Return trip needed |
| startTime / endTime | datetime | Optional | Delivery slot (Tehran time) |
| podEnabled / popEnabled | bool | Optional | Proof of delivery / pickup |

<h4>Terminal (request, inside terminals[])</h4>
| Field | Type | Required | Description |
|---|---|---|---|
| latitude | string | Required | e.g. 35.757523 |
| longitude | string | Required | e.g. 51.409911 |
| type | string | Required | pickup or drop |
| reference | string | Required | Numeric string, defines stop order, e.g. "1" |
| phoneNumber | string | Required | 11-digit phone, e.g. 09121112233 |
| address | string | Optional | Full address text |
| contactName | string | Optional | Contact person |
| comment | string | Optional | Notes for driver |

<h4>OrderDetail v1 (response — GET /v1/orders/:orderId and /references/:refId)</h4>
| Field | Type | Nullable | Description |
|---|---|---|---|
| orderId, refId, status, deliveryCategory, paymentType, trackingUrl, cancelledBy, batchIn | string | Some | Core order fields; status uses Order Status Values |
| createdTime, updatedTime, reservationDate | string | Yes | Timestamps |
| price | number | No | Order price |
| canCancel | bool | No | Whether cancel is currently allowed |
| scheduling, hasReturn | bool | Yes | Flags |
| terminals | TerminalResponse[] | No | Stops with per-terminal status (Terminal Status Values), id, coordinates, contact info, hasPickupCode, code |
| items | Item[] | No | name, quantity, weight, volume, packageValue, quantityMeasuringUnit |
| allotment | Allotment | No | Driver info: name, phoneNumber, vehiclePlateNumber, imageUrl, Status |

<h4>OrderListResponse (response — GET /v1/orders/list)</h4>
Array of order detail objects (`id`, `customerRefId`, `status`, `price`, `trackingUrl`, `createdAt`, `terminals`, `itemDetails`, ...). Tolerate additional fields.

<h4>OrderLocationResponse (response)</h4>
| Field | Type | Nullable | Description |
|---|---|---|---|
| latitude | number | No | Driver latitude |
| longitude | number | No | Driver longitude |

<h4>EventResponse (response — array)</h4>
| Field | Type | Nullable | Description |
|---|---|---|---|
| type | string | No | Event type |
| actionAt | datetime | No | When it happened |

<h4>OrderVerifyRequest (request)</h4>
| Field | Type | Required | Description |
|---|---|---|---|
| orderId | string | Required | Platform order ID |
| customerRefId | string | Required | Your refId |
| verificationType | string | Required | ARRIVED_AT_PICKUP, PICKED_UP, or READY_FOR_PICKUP |

<h4>VerifyPickupCodeRequest (request)</h4>
| Field | Type | Required | Description |
|---|---|---|---|
| pickupCode | string | Required | Pickup code handed to driver |
| terminalId | string | Required | Terminal ID of pickup terminal |
| customerRefId | string | Required | Your refId |
| orderId | string | Required only if you don't send customerRefId | Box order ID |
| referenceOrderId | string | Optional | Parent order ID of batched orders |

<h4>NumberMaskingRequest (request)</h4>
| Field | Type | Required | Description |
|---|---|---|---|
| orderId | string | Required | Platform order ID |
| terminalId | int | Required | Terminal ID |

#### 5.3 Orders v2 (Classic + Same-day)
<h4>OrderCreateRequest v2 (request)</h4>
| Field | Type | Required | Description |
|---|---|---|---|
| refId | string | Optional (Required for Same-day) | Your unique business reference |
| terminals | Terminal[] | Required | Pickup + dropoff stops |
| deliveryCategory | string | Required | same-day for Same-day; other value = Classic |
| paymentType | string | Required | e.g. prepaid |
| city | string | Required | e.g. tehran |
| timeSlot | object | Optional (Required for Same-day) | Must include id for Same-day (from sdd/timelines) |
| packages | array | Optional (Required for Same-day) | Items being delivered |
| features | object | Optional | Extra order features |
| totalPackageSize, waitingTime, voucherCode, hasReturn, startTime, endTime, podEnabled, popEnabled | mixed | Optional | Same meaning as v1 |

<h4>CreateOrderResponse v2 (response, 201)</h4>
| Field | Type | Nullable | Description |
|---|---|---|---|
| orderId, refId, status | string | No | Core identifiers; store orderId |
| price | number | No | Order price |
| createdTime | string | Yes | Creation time |
| terminalIds | int[] | No | IDs of created terminals |
| packages | PackageResponse[] | No | Each has verification_code, pod_code, tracking_code (all nullable) |

<h4>OrderDetail v2 (response)</h4>
Same shape as v1 OrderDetail (`orderId`, `refId`, `status`, `deliveryCategory`, `paymentType`, `price`, `canCancel`, `trackingUrl`, `createdTime`, `updatedTime`, `terminals`, `items`, `allotment`). Several fields nullable — tolerate nulls.

#### 5.4 NXP
<h4>Size object (used in requests)</h4>
| Field | Type | Required | Description |
|---|---|---|---|
| type | string | Required | category or dimension |
| value | string | Required | category → size ID (e.g. "3"); dimension → "L,W,H" (e.g. "10,20,30") |

<h4>ParcelSize (response — GET /v1/nxp/sizes/:delivery_type, array)</h4>
| Field | Type | Nullable | Description |
|---|---|---|---|
| id, capacity | int | No | Size ID and capacity |
| key, title, delivery_type, description | string | No | Size identifiers/labels |
| width, height, depth | number | No | Dimensions |

<h4>SearchTimelinesRequest (request — POST /v1/nxp/timelines)</h4>
| Field | Type | Required | Description |
|---|---|---|---|
| parcels | Parcel[] | Required | At least 1 parcel |

Parcel: `size` (Size, required), `sender_terminal_id` (string, required, format `fav_id:<id>`), and either `receiver_terminal` (object) OR `receiver_terminal_id` (string) — one of the two is required.

receiver_terminal object: `address` (required), `cellphone` (required), `city_id` (int, required, > 0); optional: `name`, `latitude`, `longitude`, `postal_code`.

<h4>TimelineResponse (response — array)</h4>
| Field | Type | Nullable | Description |
|---|---|---|---|
| id | string | No | Timeline ID — store it, needed for invoice/create |
| pickup_timeslot, delivery_timeslot | Timeslot | No | id, hub_id, day, start_time, end_time, city_id |

<h4>SddTimelineRequest (request — POST /v1/nxp/sdd/timelines)</h4>
| Field | Type | Required | Description |
|---|---|---|---|
| parcels | array | Required | At least 1; each has a required size |
| sender_terminal_id | string | Required | Favorite Address ID, format fav_id:<id> |

<h4>SddTimelineResponse (response — array)</h4>
| Field | Type | Nullable | Description |
|---|---|---|---|
| id | string | No | Time slot ID — use as timeSlot.id in POST /v2/orders |
| status, day, start_time, end_time | string | No | Slot details |
| reserved_count | int | No | Existing reservations |

<h4>BundleResponse (response — GET /v1/nxp/bundles, array)</h4>
| Field | Type | Nullable | Description |
|---|---|---|---|
| code, created_at, updated_at | string | No | Bundle code and timestamps |
| parcels_count, active_parcels_count | int | No | Parcel counts |
| pickup_timeslot | Timeslot | No | Pickup slot of the bundle |
| sender_terminal | TerminalResponse | No | Pickup terminal incl. fav_id |

<h4>OrderInvoiceRequest / CreateOrderReq (request — invoice and create share the parcel shape)</h4>
| Field | Type | Required | Description |
|---|---|---|---|
| invoice_token | string | Required (create only) | Token from invoice response |
| parcels | CreateParcelReq[] | Required | At least 1 parcel |
| reference | string | Optional | Your reference, 1–50 chars |

CreateParcelReq: `timeline_id` (required, from timelines step), `sender_terminal_id` (required, `fav_id:<id>`), `size` (Size, required), `receiver_terminal` or `receiver_terminal_id` (one required); optional: `pickup_type` (`snapp-box` default or `drop_off`), `value` (int, >= 0), `breakable`, `packaging` (bool, default false).

<h4>OrderInvoiceResponse (response)</h4>
| Field | Type | Nullable | Description |
|---|---|---|---|
| total, base_price, first_mile, mid_mile, last_mile, discount, insurance, tax | int | No | Price breakdown |
| reference | string | No | Your reference |
| token | string | No | Store this — required as invoice_token in create order |

<h4>CreateSenderOrderResponse / FindOrdersResponse (response — create 201, search 200 array)</h4>
| Field | Type | Nullable | Description |
|---|---|---|---|
| id, status | string | No | Order ID (store it) and status |
| reference | string | Yes | Your reference |
| parcels_count | int | No | Parcel count |
| amount | int | No | Total amount |
| created_at, updated_at | datetime | No | Timestamps |
| tags | object | No | pickup_timeslots, delivery_timeslots, sender_terminals, pickup_types |
| timelines | array | No | id, status, order_cutoff |

<h4>PaymentResponse (response — POST /v1/nxp/orders/:orderId/transactions)</h4>
| Field | Type | Nullable | Description |
|---|---|---|---|
| url | string | No | Payment URL (if any) |
| paid | bool | No | Whether payment completed |

<h4>FindParcelsResponse (response — GET /v1/nxp/parcels, array)</h4>
Per parcel: `id`, `order_id`, `timeline_id`, `tracking_code`, `status`, `reference`, `verification_code` (int), `value` (int), `breakable` (bool), `created_at`/`updated_at`, `weight` (`value`, `category`), `size` (`value`, `category`), `sender_terminal`/`receiver_terminal` (objects, nullable), `pickup_timeslot`/`delivery_timeslot` (nullable), `latest_trip` (nullable, includes driver info). Tolerate nulls and extra fields.

<h4>TransactionResponse (response — GET /v1/nxp/transactions, array)</h4>
| Field | Type | Nullable | Description |
|---|---|---|---|
| transaction_id, status, type, gateway | string | No | Transaction details |
| paid_amount, remaining, total_amount | int | No | Amounts |
| finalized_at | datetime | No | Finalization time |

<h4>ParcelCancelRequest (request — DELETE /v1/nxp/parcels/:id)</h4>
| Field | Type | Required | Description |
|---|---|---|---|
| reason_id | int | Required | Cancellation reason ID |

#### 5.5 Customers, Wallet, Insurance
<h4>FavoriteAddress (request POST, response 201; GET returns array)</h4>
| Field | Type | Required | Description |
|---|---|---|---|
| name | string | Required | Label, e.g. "Office" |
| contactName | string | Required | Contact person |
| contactPhoneNumber | string | Required | 11-digit phone, e.g. 09121234567 |
| latitude, longitude | string | Required | Coordinates |
| address | string | Required | Full address text |
| plate, unit, comment | string | Optional | Extra location details |
| defaultAddress | bool | Optional | Set as default |

Response adds `id` (store it — needed for `fav_id:<id>`), `usageCount`, and nullable `imageUrl`, `useCase`, `status`.

<h4>BalanceResponse (response — GET /v1/wallets)</h4>
| Field | Type | Nullable | Description |
|---|---|---|---|
| balance | number | Yes | Current wallet balance |
| currency | string | No | e.g. IRR |

<h4>GetInsuranceResponse (response — GET /v1/insurances)</h4>
| Field | Type | Nullable | Description |
|---|---|---|---|
| city, deliveryCategory | string | No | Scope of the options |
| packageValues | array | No | Each: id (int), title (string), value (number), insuranceAmount (number). Use id as insuranceId in order packages. |

<h4>ErrorResponse (response — all failed calls)</h4>
| Field | Type | Nullable | Description |
|---|---|---|---|
| key | string | No | Machine-readable error key, e.g. invalid.request.payload |
| message | string | No | Human-readable message |
| statusCode | string | No | HTTP status as string, e.g. "400" |

### 6. Order Flows

#### 6.1 Classic Order Flow (v1)
1. GET /v1/orders/delivery-categories
2. POST /v1/pricing
3. POST /v1/orders
4. GET /v1/orders/:id
5. GET /v1/orders/:id/events

| Step | Why | Data carried forward | Failure modes |
|---|---|---|---|
| 1. Delivery categories | Discover valid deliveryCategory values for the city/location. All query params mandatory. | Chosen deliveryCategory | 400 if latitude/longitude missing |
| 2. Pricing | Quote the price before creating. Optional but strongly recommended. | Quoted price (display to your customer) | 400 on invalid body |
| 3. Create order | Create the order. | Store orderId and your refId — needed for all later calls | 400 on missing required fields |
| 4. Order detail | Poll status, get driver info, tracking URL. | status, terminal statuses, canCancel | 404 if wrong orderId |
| 5. Events | Full history of what happened. | — | 404 if wrong orderId |

Additional operations: update with `PUT /v1/orders/:orderId` (only after `ACCEPTED` and just before `PICKED_UP`), cancel with `DELETE /v1/orders/:orderId`, verify events with `POST /v1/orders/verify` or `POST /v1/orders/verify/picked-up/:refId`, verify pickup code with `POST /v1/customers/verify-pickup-code`.

#### 6.2 Same-day Order Flow (v2)
1. POST /v1/customers/favorite-address
2. GET /v1/nxp/sizes/same-day
3. POST /v1/nxp/sdd/timelines
4. POST /v2/orders

| Step | Why | Data carried forward |
|---|---|---|
| 1. Create Favorite Address | The pickup location must be a saved Favorite Address. | Favorite Address id |
| 2. Get sizes | Available sizes for Same-day delivery. | Chosen size |
| 3. Get SDD timelines | Available same-day time slots. Send sender_terminal_id as fav_id:<id>. | Chosen slot id |
| 4. Create order | POST /v2/orders with deliveryCategory: "same-day", refId, timeSlot.id, and packages (all required for Same-day). | Store orderId |

Track afterwards with `GET /v2/orders/:orderId` or `GET /v2/orders/references/:refId`.

#### 6.3 NXP Order Flow

:::note
Pre-action: before creating an NXP order, save the pickup terminal as a Favorite Address (POST /v1/customers/favorite-address). Use the saved Favorite Address ID as the pickup terminal ID in all NXP calls.
:::

1. GET /v1/nxp/sizes/:type
2. POST /v1/nxp/timelines
3. GET /v1/nxp/bundles (optional)
4. POST /v1/nxp/orders/invoice
5. POST /v1/nxp/orders
6. POST /v1/nxp/orders/:id/transactions

| Step | Why | Data carried forward |
|---|---|---|
| 1. Sizes | Get available parcel sizes. | Chosen size (type/value) |
| 2. Timelines | Get pickup/delivery time slots. | timeline_id |
| 3. Bundles (optional) | If you already have an order in ON_GOING status, list bundles to pick the pickup date of your new order based on the existing ON_GOING bundle. Requires bundle_status and sender_terminal_id query params. | Pickup date decision |
| 4. Invoice | Price quote. | token → invoice_token |
| 5. Create order | Create with invoice_token + timeline_id. Send unique X-Idempotency-Key header so retries never create duplicate orders. | Order id |
| 6. Pay | Pay from your Snapp Box wallet. Check balance first with GET /v1/wallets. | paid flag |

Afterwards: search orders (`GET /v1/nxp/orders`), parcels (`GET /v1/nxp/parcels`), labels (`GET /v1/nxp/parcels/labels`, returns HTML), cancel parcel (`DELETE /v1/nxp/parcels/:id`), list transactions (`GET /v1/nxp/transactions`).

### 7. Endpoint-by-Endpoint Implementation Checklist
| Endpoint | Method | Purpose | Request DTO | Response DTO | Required? | Important Validation |
|---|---|---|---|---|---|---|
| /v1/oauth2/token | POST | Get access token | TokenRequest | TokenResponse | Required (all flows) | client_id + client_secret required |
| /v1/orders/delivery-categories | GET | List delivery categories | — | DeliveryCategoriesResponse | Classic flow step 1 | ALL query params mandatory (latitude, longitude) |
| /v1/pricing | POST | Price quote (v1) | OrderCreateRequest-like | PricingResponse | Recommended (Classic) | Same required fields as create |
| /v1/orders | POST | Create Classic order | OrderCreateRequest v1 | Order detail (201) | Classic flow step 3 | refId, terminals, deliveryCategory, paymentType, city required |
| /v1/orders/:orderId | GET | Order detail | — | OrderDetail v1 | Required (tracking) | — |
| /v1/orders/references/:refId | GET | Order detail by refId | — | OrderDetail v1 | Optional | — |
| /v1/orders/list | GET | List orders | — | OrderListResponse | Optional | — |
| /v1/orders/:orderId/current-location | GET | Live driver location | — | OrderLocationResponse | Optional | — |
| /v1/orders/:orderId/events | GET | Event history | — | EventResponse[] | Optional | — |
| /v1/orders/:orderId | PUT | Update order | Update body | 204 no body | Optional | Only after ACCEPTED, before PICKED_UP |
| /v1/orders/:orderId | DELETE | Cancel order | — | 204 no body | Optional | — |
| /v1/orders/verify | POST | Verify order event | OrderVerifyRequest | 204 no body | Optional | verificationType enum: ARRIVED_AT_PICKUP, PICKED_UP, READY_FOR_PICKUP |
| /v1/orders/verify/picked-up/:refId | POST | Verify picked-up by refId | — | 204 no body | Optional | — |
| /v1/customers/verify-pickup-code | POST | Verify pickup code | VerifyPickupCodeRequest | 200 | Optional | pickupCode, terminalId, customerRefId required; orderId required if no customerRefId |
| /v1/orders/number-masking | POST | Mask phone numbers | NumberMaskingRequest | 204 no body | Optional | orderId + terminalId required |
| /v2/orders | POST | Create order (Classic or Same-day) | OrderCreateRequest v2 | CreateOrderResponse v2 (201) | v2 / Same-day flow | terminals, deliveryCategory, paymentType, city required; refId, timeSlot.id, packages required for Same-day |
| /v2/orders/:orderId | GET | Order detail v2 | — | OrderDetail v2 | v2 flow | — |
| /v2/orders/references/:refId | GET | Order detail by refId | — | OrderDetail v2 | Optional | — |
| /v1/customers/favorite-address | POST | Save favorite address | FavoriteAddress | FavoriteAddress (201) | NXP + Same-day pre-action | name, contactName, contactPhoneNumber, latitude, longitude, address required |
| /v1/customers/favorite-address | GET | List favorite addresses | — | FavoriteAddress[] | Optional | — |
| /v1/nxp/sizes/:delivery_type | GET | Parcel sizes | — | ParcelSize[] | NXP step 1 / Same-day step 2 | delivery_type: standard or same-day |
| /v1/nxp/timelines | POST | Search time slots | SearchTimelinesRequest | TimelineResponse[] | NXP step 2 | parcels required; receiver_terminal OR receiver_terminal_id |
| /v1/nxp/sdd/timelines | POST | Same-day time slots | SddTimelineRequest | SddTimelineResponse[] | Same-day step 3 | parcels + sender_terminal_id (fav_id:<id>) required |
| /v1/nxp/bundles | GET | List bundles | — | BundleResponse[] | Optional (NXP step 3) | bundle_status + sender_terminal_id query params required |
| /v1/nxp/orders/invoice | POST | Price quote | OrderInvoiceRequest | OrderInvoiceResponse | NXP step 4 | parcels required |
| /v1/nxp/orders | POST | Create NXP order | CreateOrderReq | CreateSenderOrderResponse (201) | NXP step 5 | invoice_token + parcels required; X-Idempotency-Key header |
| /v1/nxp/orders/:orderId/transactions | POST | Pay order | — | PaymentResponse | NXP step 6 | — |
| /v1/nxp/orders | GET | Search orders | — | FindOrdersResponse[] | Optional | — |
| /v1/nxp/parcels | GET | Search parcels | — | FindParcelsResponse[] | Optional | — |
| /v1/nxp/parcels/labels | GET | Parcel labels (HTML) | — | HTML page | Optional | Response is text/html, not JSON |
| /v1/nxp/parcels/:id | DELETE | Cancel parcel | ParcelCancelRequest | 204 no body | Optional | reason_id required |
| /v1/nxp/transactions | GET | List transactions | — | TransactionResponse[] | Optional | — |
| /v1/wallets | GET | Wallet balance | — | BalanceResponse | Recommended before NXP payment | — |
| /v1/insurances | GET | Insurance options | — | GetInsuranceResponse | Optional | — |

### 8. Client-Side Validation
Validate before sending. Failing fast locally avoids a wasted round-trip and a 400.

| Check | Rule |
|---|---|
| Required fields | Every field marked Required in section 5 must be present and non-empty. |
| Phone numbers | 11 digits only, e.g. 09121112233. Applies to phoneNumber, contactPhoneNumber, cellphone. |
| Enums | deliveryCategory must come from the delivery-categories call; verificationType must be ARRIVED_AT_PICKUP, PICKED_UP, or READY_FOR_PICKUP; Terminal type is pickup or drop. |
| Either/or fields | NXP parcels: exactly one of receiver_terminal or receiver_terminal_id. |
| Conditional requirements | Same-day orders: refId, timeSlot.id, and packages all required. VerifyPickupCode: orderId required when customerRefId is absent. |
| Formats | sender_terminal_id must be fav_id:<id>. Date/time fields ISO-8601 with Asia/Tehran offset. NXP Size value is a size ID for category type, or "L,W,H" for dimension type. |
| Numeric constraints | NXP city_id > 0; parcel value >= 0; reference 1–50 chars. |

### 9. Idempotency & Safe Retries of Creates

- Orders v1/v2: send a **unique `refId` per business order**. Retrying a create with the same `refId` is safe.

- NXP create: send a unique `X-Idempotency-Key` header so retries never create duplicate orders.

- Never invent a new `refId` / idempotency key on retry — reuse the original or you will create duplicates.

### 10. Error Handling
Every failed call returns the same shape: `key`, `message`, `statusCode` (see [Errors & Conventions](#errors)). Map them to your domain errors:

| Status | Meaning | Client action |
|---|---|---|
| 400 | Missing/invalid fields | Log the key, fix the request, do not retry unchanged. |
| 401 | Missing/expired token | Request a new token from /v1/oauth2/token, retry the call once with the new token. |
| 404 | Wrong ID in path | Fail; check stored orderId/refId. |
| 500 | Server/upstream failure | Retry with backoff (section 11); escalate to SnappBox support if persistent. |
| Network/timeout | No response received | Retry reads freely. Retry creates only with the original refId / X-Idempotency-Key (section 9). |

### 11. Retry Strategy
No retry policy is documented by the API — the following is a recommended client-side default:

- **Reads** (GET): retry on timeout, 500, and 429 with exponential backoff (e.g. 1s, 2s, 4s, max 3–5 attempts).

- **Creates** (POST orders): retry only with the original idempotency key (`refId` / `X-Idempotency-Key`) — otherwise a retry after a timeout may double-create.

- **401**: refresh token once, then retry once. A second 401 means credentials are wrong — fail loudly.

- **400**: never retry without changing the request.

### 12. Token Lifecycle

- Fetch a token from `POST /v1/oauth2/token` before the first protected call.

- Cache it; reuse until it expires (`expires_in` seconds).

- Proactively refresh shortly before expiry, or reactively on the first 401.

- Serialize refreshes: concurrent 401s should trigger one refresh, not many.

### 13. Testing Checklist
| Scenario | Expected result |
|---|---|
| Create order with a missing required field | Local validation rejects; no HTTP call made |
| Create order, then retry with same refId | No duplicate order created |
| Call with expired token | 401 → token refresh → retry succeeds |
| GET detail with unknown orderId | 404 mapped to your not-found error |
| Response containing unknown extra fields | Deserializer ignores them, no crash |
| Nullable fields returned as null | Deserializer tolerates nulls |
| Same-day create without timeSlot.id | Local validation rejects |
| NXP create retried with same X-Idempotency-Key | No duplicate order |

### 14. Documentation Gaps
Not documented by the API — decide client-side or confirm with SnappBox support:

- **Base URL** — obtain from SnappBox Support; never hardcode from docs.

- **HTTP timeout values** — pick client-side defaults (e.g. 10s connect / 30s read).

- **Retry policy / rate limits** — no documented limits; use the backoff in section 11.

- **Environments** — staging vs. production URLs not documented; ask support.

### 15. Logging & Monitoring

- Log every call with: endpoint, HTTP status, latency, `orderId`/`refId` when known, and a correlation ID per business operation.

- On errors, log the error `key` — it is the stable machine-readable identifier; `message` may change.

- Never log `client_secret`, access tokens, or full Authorization headers.

- Alert on: repeated 401s (credential problem), sustained 5xx (gateway problem), create retries that exhaust backoff.

- Track flow-completion metrics per flow in section 6: quote → create → paid/delivered rates surface integration breakage early.


## Implementation Plan

*Step-by-step plan for your development team to build a client-side integration with the Snapp Box B2B API. Language-agnostic — implement in any modern programming language.*

### 1. Introduction
This integration lets your system create, track, update, and cancel Snapp Box delivery orders through the B2B gateway. The gateway is the front desk of Snapp Box: your system never talks to internal services directly.

Supported services:

- **Classic** — point-to-point delivery (bike, car...) via `/v1/orders` or `/v2/orders`.

- **Same-day** — same-day delivery via `/v2/orders` with `deliveryCategory: "same-day"`.

- **NXP sender orders** — parcel shipping with timelines, invoices, and wallet payment via `/v1/nxp`.

Overall order lifecycle: quote price → create order → order moves through statuses (`PENDING` → `ACCEPTED` → `PICKED_UP` → `DELIVERED`, or `CANCELLED`) → track via detail/events endpoints.

### 2. Integration Architecture
Recommended client-side layers (any language/framework):

```
Client Application
→ API Client / HTTP Layer      (raw HTTP calls, headers, timeouts)
→ DTO / Serialization Layer    (request/response models, JSON serialize/deserialize)
→ Business Service             (order flows, sequence orchestration)
→ Validation                   (client-side checks before sending)
→ Error Handling               (map HTTP/API errors to your domain errors)
→ Logging / Monitoring         (correlation, latency, failures)
```

| Layer | Responsibility |
|---|---|
| API Client / HTTP | Send requests, attach Authorization: Bearer header, enforce timeouts, return raw responses. |
| DTO / Serialization | Define all request/response models (section 5). Serialize requests to JSON; deserialize responses. Tolerate unknown/extra fields in responses. |
| Business Service | Implement the order flows (section 6) in the documented call order. Store identifiers between steps. |
| Validation | Check required fields, formats, and enums before sending (section 8). Fail fast locally instead of getting a 400. |
| Error Handling | Parse the error response shape, decide retry vs. fail (section 10). |
| Logging / Monitoring | Correlation IDs, order IDs, status codes, latency (section 15). |

### 3. Configuration
Make these configurable (environment variables, config files, or a config service):

| Setting | Notes |
|---|---|
| Base URL | Obtain from SnappBox Support. Do not hardcode a URL from documentation. |
| client_id / client_secret | Issued credentials for POST /v1/oauth2/token. Store securely (secret manager), never in source code or logs. |
| HTTP timeout | Recommended client-side setting. No value is documented — see Documentation Gaps. |
| Retry configuration | Recommended client-side setting. Not documented — see section 11 and Documentation Gaps. |
| Environment | Separate config per environment (staging/production) if SnappBox provides multiple Base URLs. |

### 4. API Client Layer
Rules that apply to every endpoint:

- All endpoints except `POST /v1/oauth2/token` require header `Authorization: Bearer <access_token>`.

- Requests with a body need `Content-Type: application/json`.

- Date/time fields use **Asia/Tehran**, ISO-8601 with offset (e.g. `2025-08-06T19:00:00+03:30`).

- Fields marked **Required** must be present or the gateway returns 400.

Endpoint-by-endpoint details (method, path, params, bodies, statuses) are in the endpoint pages: [Orders v1](#orders), [Orders v2](#orders-v2), [NXP](#nxp), [Customers](#customers), [Wallets & Insurance](#wallets), [Authentication](#auth). The checklist in section 7 lists every endpoint you must implement.

Implementation requirements per endpoint:

- Build URLs from configurable Base URL + documented path; substitute path parameters (`:orderId`, `:refId`, `:delivery_type`, `:id`).

- Serialize query parameters exactly as documented; note that `GET /v1/orders/delivery-categories` requires **all** query parameters, and `GET /v1/nxp/bundles` requires `bundle_status` and `sender_terminal_id`.

- Expect documented success statuses: 200 for reads/actions, 201 for order/address creation, 204 (no body) for update/cancel/verify/number-masking.

### 5. DTO / Data Model Implementation
Implement every model below. Field names are JSON keys exactly as the API expects/returns them. "Nullable" means the field may be null or absent — your deserializer must tolerate both.

#### 5.1 Authentication
<h4>TokenRequest (request)</h4>
| Field | Type | Required | Description |
|---|---|---|---|
| client_id | string | Required | Issued client ID |
| client_secret | string | Required | Issued client secret |

<h4>TokenResponse (response)</h4>
| Field | Type | Nullable | Description |
|---|---|---|---|
| access_token | string | No | JWT to send as Bearer token |
| token_type | string | No | Bearer |
| expires_in | number | No | Token lifetime in seconds; when you get a 401, request a new token and retry |

#### 5.2 Orders v1 (Classic)
<h4>DeliveryCategoriesResponse (response)</h4>
| Field | Type | Nullable | Description |
|---|---|---|---|
| city | string | No | City the categories apply to |
| deliveryCategories | string[] | No | Valid deliveryCategory values, e.g. bike, bike-without-box |

<h4>PricingResponse (response)</h4>
| Field | Type | Nullable | Description |
|---|---|---|---|
| totalFare | int | No | Total price |
| finalCustomerFare | int | No | Price charged to customer |
| distance | number | No | Route distance |
| eta | number | No | Estimated minutes |
| pricingId | string | Yes | Quote identifier |
| onDemandBiddingEnabled / scheduledBiddingEnabled | bool | No | Bidding flags |

:::note
PricingResponse contains additional optional pricing fields (subsidy, surge, voucher info...). Treat all unlisted fields as optional and ignore unknown fields.
:::

<h4>OrderCreateRequest v1 (request)</h4>
| Field | Type | Required | Description |
|---|---|---|---|
| refId | string | Required | Your unique business reference (idempotency key) |
| terminals | Terminal[] | Required | At least one pickup and one drop |
| deliveryCategory | string | Required | From delivery-categories step |
| paymentType | string | Required | e.g. prepaid |
| city | string | Required | e.g. tehran |
| packages | array | Optional | Items being delivered |
| totalPackageSize | number | Optional | Total size |
| waitingTime | int | Optional | Minutes driver waits |
| voucherCode | string | Optional | Discount code |
| hasReturn | bool | Optional | Return trip needed |
| startTime / endTime | datetime | Optional | Delivery slot (Tehran time) |
| podEnabled / popEnabled | bool | Optional | Proof of delivery / pickup |

<h4>Terminal (request, inside terminals[])</h4>
| Field | Type | Required | Description |
|---|---|---|---|
| latitude | string | Required | e.g. 35.757523 |
| longitude | string | Required | e.g. 51.409911 |
| type | string | Required | pickup or drop |
| reference | string | Required | Numeric string, defines stop order, e.g. "1" |
| phoneNumber | string | Required | 11-digit phone, e.g. 09121112233 |
| address | string | Optional | Full address text |
| contactName | string | Optional | Contact person |
| comment | string | Optional | Notes for driver |

<h4>OrderDetail v1 (response — GET /v1/orders/:orderId and /references/:refId)</h4>
| Field | Type | Nullable | Description |
|---|---|---|---|
| orderId, refId, status, deliveryCategory, paymentType, trackingUrl, cancelledBy, batchIn | string | Some | Core order fields; status uses Order Status Values |
| createdTime, updatedTime, reservationDate | string | Yes | Timestamps |
| price | number | No | Order price |
| canCancel | bool | No | Whether cancel is currently allowed |
| scheduling, hasReturn | bool | Yes | Flags |
| terminals | TerminalResponse[] | No | Stops with per-terminal status (Terminal Status Values), id, coordinates, contact info, hasPickupCode, code |
| items | Item[] | No | name, quantity, weight, volume, packageValue, quantityMeasuringUnit |
| allotment | Allotment | No | Driver info: name, phoneNumber, vehiclePlateNumber, imageUrl, Status |

<h4>OrderListResponse (response — GET /v1/orders/list)</h4>
Array of order detail objects (`id`, `customerRefId`, `status`, `price`, `trackingUrl`, `createdAt`, `terminals`, `itemDetails`, ...). Tolerate additional fields.

<h4>OrderLocationResponse (response)</h4>
| Field | Type | Nullable | Description |
|---|---|---|---|
| latitude | number | No | Driver latitude |
| longitude | number | No | Driver longitude |

<h4>EventResponse (response — array)</h4>
| Field | Type | Nullable | Description |
|---|---|---|---|
| type | string | No | Event type |
| actionAt | datetime | No | When it happened |

<h4>OrderVerifyRequest (request)</h4>
| Field | Type | Required | Description |
|---|---|---|---|
| orderId | string | Required | Platform order ID |
| customerRefId | string | Required | Your refId |
| verificationType | string | Required | ARRIVED_AT_PICKUP, PICKED_UP, or READY_FOR_PICKUP |

<h4>VerifyPickupCodeRequest (request)</h4>
| Field | Type | Required | Description |
|---|---|---|---|
| pickupCode | string | Required | Pickup code handed to driver |
| terminalId | string | Required | Terminal ID of pickup terminal |
| customerRefId | string | Required | Your refId |
| orderId | string | Required only if you don't send customerRefId | Box order ID |
| referenceOrderId | string | Optional | Parent order ID of batched orders |

<h4>NumberMaskingRequest (request)</h4>
| Field | Type | Required | Description |
|---|---|---|---|
| orderId | string | Required | Platform order ID |
| terminalId | int | Required | Terminal ID |

#### 5.3 Orders v2 (Classic + Same-day)
<h4>OrderCreateRequest v2 (request)</h4>
| Field | Type | Required | Description |
|---|---|---|---|
| refId | string | Optional (Required for Same-day) | Your unique business reference |
| terminals | Terminal[] | Required | Pickup + dropoff stops |
| deliveryCategory | string | Required | same-day for Same-day; other value = Classic |
| paymentType | string | Required | e.g. prepaid |
| city | string | Required | e.g. tehran |
| timeSlot | object | Optional (Required for Same-day) | Must include id for Same-day (from sdd/timelines) |
| packages | array | Optional (Required for Same-day) | Items being delivered |
| features | object | Optional | Extra order features |
| totalPackageSize, waitingTime, voucherCode, hasReturn, startTime, endTime, podEnabled, popEnabled | mixed | Optional | Same meaning as v1 |

<h4>CreateOrderResponse v2 (response, 201)</h4>
| Field | Type | Nullable | Description |
|---|---|---|---|
| orderId, refId, status | string | No | Core identifiers; store orderId |
| price | number | No | Order price |
| createdTime | string | Yes | Creation time |
| terminalIds | int[] | No | IDs of created terminals |
| packages | PackageResponse[] | No | Each has verification_code, pod_code, tracking_code (all nullable) |

<h4>OrderDetail v2 (response)</h4>
Same shape as v1 OrderDetail (`orderId`, `refId`, `status`, `deliveryCategory`, `paymentType`, `price`, `canCancel`, `trackingUrl`, `createdTime`, `updatedTime`, `terminals`, `items`, `allotment`). Several fields nullable — tolerate nulls.

#### 5.4 NXP
<h4>Size object (used in requests)</h4>
| Field | Type | Required | Description |
|---|---|---|---|
| type | string | Required | category or dimension |
| value | string | Required | category → size ID (e.g. "3"); dimension → "L,W,H" (e.g. "10,20,30") |

<h4>ParcelSize (response — GET /v1/nxp/sizes/:delivery_type, array)</h4>
| Field | Type | Nullable | Description |
|---|---|---|---|
| id, capacity | int | No | Size ID and capacity |
| key, title, delivery_type, description | string | No | Size identifiers/labels |
| width, height, depth | number | No | Dimensions |

<h4>SearchTimelinesRequest (request — POST /v1/nxp/timelines)</h4>
| Field | Type | Required | Description |
|---|---|---|---|
| parcels | Parcel[] | Required | At least 1 parcel |

Parcel: `size` (Size, required), `sender_terminal_id` (string, required, format `fav_id:<id>`), and either `receiver_terminal` (object) OR `receiver_terminal_id` (string) — one of the two is required.

receiver_terminal object: `address` (required), `cellphone` (required), `city_id` (int, required, > 0); optional: `name`, `latitude`, `longitude`, `postal_code`.

<h4>TimelineResponse (response — array)</h4>
| Field | Type | Nullable | Description |
|---|---|---|---|
| id | string | No | Timeline ID — store it, needed for invoice/create |
| pickup_timeslot, delivery_timeslot | Timeslot | No | id, hub_id, day, start_time, end_time, city_id |

<h4>SddTimelineRequest (request — POST /v1/nxp/sdd/timelines)</h4>
| Field | Type | Required | Description |
|---|---|---|---|
| parcels | array | Required | At least 1; each has a required size |
| sender_terminal_id | string | Required | Favorite Address ID, format fav_id:<id> |

<h4>SddTimelineResponse (response — array)</h4>
| Field | Type | Nullable | Description |
|---|---|---|---|
| id | string | No | Time slot ID — use as timeSlot.id in POST /v2/orders |
| status, day, start_time, end_time | string | No | Slot details |
| reserved_count | int | No | Existing reservations |

<h4>BundleResponse (response — GET /v1/nxp/bundles, array)</h4>
| Field | Type | Nullable | Description |
|---|---|---|---|
| code, created_at, updated_at | string | No | Bundle code and timestamps |
| parcels_count, active_parcels_count | int | No | Parcel counts |
| pickup_timeslot | Timeslot | No | Pickup slot of the bundle |
| sender_terminal | TerminalResponse | No | Pickup terminal incl. fav_id |

<h4>OrderInvoiceRequest / CreateOrderReq (request — invoice and create share the parcel shape)</h4>
| Field | Type | Required | Description |
|---|---|---|---|
| invoice_token | string | Required (create only) | Token from invoice response |
| parcels | CreateParcelReq[] | Required | At least 1 parcel |
| reference | string | Optional | Your reference, 1–50 chars |

CreateParcelReq: `timeline_id` (required, from timelines step), `sender_terminal_id` (required, `fav_id:<id>`), `size` (Size, required), `receiver_terminal` or `receiver_terminal_id` (one required); optional: `pickup_type` (`snapp-box` default or `drop_off`), `value` (int, >= 0), `breakable`, `packaging` (bool, default false).

<h4>OrderInvoiceResponse (response)</h4>
| Field | Type | Nullable | Description |
|---|---|---|---|
| total, base_price, first_mile, mid_mile, last_mile, discount, insurance, tax | int | No | Price breakdown |
| reference | string | No | Your reference |
| token | string | No | Store this — required as invoice_token in create order |

<h4>CreateSenderOrderResponse / FindOrdersResponse (response — create 201, search 200 array)</h4>
| Field | Type | Nullable | Description |
|---|---|---|---|
| id, status | string | No | Order ID (store it) and status |
| reference | string | Yes | Your reference |
| parcels_count | int | No | Parcel count |
| amount | int | No | Total amount |
| created_at, updated_at | datetime | No | Timestamps |
| tags | object | No | pickup_timeslots, delivery_timeslots, sender_terminals, pickup_types |
| timelines | array | No | id, status, order_cutoff |

<h4>PaymentResponse (response — POST /v1/nxp/orders/:orderId/transactions)</h4>
| Field | Type | Nullable | Description |
|---|---|---|---|
| url | string | No | Payment URL (if any) |
| paid | bool | No | Whether payment completed |

<h4>FindParcelsResponse (response — GET /v1/nxp/parcels, array)</h4>
Per parcel: `id`, `order_id`, `timeline_id`, `tracking_code`, `status`, `reference`, `verification_code` (int), `value` (int), `breakable` (bool), `created_at`/`updated_at`, `weight` (`value`, `category`), `size` (`value`, `category`), `sender_terminal`/`receiver_terminal` (objects, nullable), `pickup_timeslot`/`delivery_timeslot` (nullable), `latest_trip` (nullable, includes driver info). Tolerate nulls and extra fields.

<h4>TransactionResponse (response — GET /v1/nxp/transactions, array)</h4>
| Field | Type | Nullable | Description |
|---|---|---|---|
| transaction_id, status, type, gateway | string | No | Transaction details |
| paid_amount, remaining, total_amount | int | No | Amounts |
| finalized_at | datetime | No | Finalization time |

<h4>ParcelCancelRequest (request — DELETE /v1/nxp/parcels/:id)</h4>
| Field | Type | Required | Description |
|---|---|---|---|
| reason_id | int | Required | Cancellation reason ID |

#### 5.5 Customers, Wallet, Insurance
<h4>FavoriteAddress (request POST, response 201; GET returns array)</h4>
| Field | Type | Required | Description |
|---|---|---|---|
| name | string | Required | Label, e.g. "Office" |
| contactName | string | Required | Contact person |
| contactPhoneNumber | string | Required | 11-digit phone, e.g. 09121234567 |
| latitude, longitude | string | Required | Coordinates |
| address | string | Required | Full address text |
| plate, unit, comment | string | Optional | Extra location details |
| defaultAddress | bool | Optional | Set as default |

Response adds `id` (store it — needed for `fav_id:<id>`), `usageCount`, and nullable `imageUrl`, `useCase`, `status`.

<h4>BalanceResponse (response — GET /v1/wallets)</h4>
| Field | Type | Nullable | Description |
|---|---|---|---|
| balance | number | Yes | Current wallet balance |
| currency | string | No | e.g. IRR |

<h4>GetInsuranceResponse (response — GET /v1/insurances)</h4>
| Field | Type | Nullable | Description |
|---|---|---|---|
| city, deliveryCategory | string | No | Scope of the options |
| packageValues | array | No | Each: id (int), title (string), value (number), insuranceAmount (number). Use id as insuranceId in order packages. |

<h4>ErrorResponse (response — all failed calls)</h4>
| Field | Type | Nullable | Description |
|---|---|---|---|
| key | string | No | Machine-readable error key, e.g. invalid.request.payload |
| message | string | No | Human-readable message |
| statusCode | string | No | HTTP status as string, e.g. "400" |

### 6. Order Flows

#### 6.1 Classic Order Flow (v1)
1. GET /v1/orders/delivery-categories
2. POST /v1/pricing
3. POST /v1/orders
4. GET /v1/orders/:id
5. GET /v1/orders/:id/events

| Step | Why | Data carried forward | Failure modes |
|---|---|---|---|
| 1. Delivery categories | Discover valid deliveryCategory values for the city/location. All query params mandatory. | Chosen deliveryCategory | 400 if latitude/longitude missing |
| 2. Pricing | Quote the price before creating. Optional but strongly recommended. | Quoted price (display to your customer) | 400 on invalid body |
| 3. Create order | Create the order. | Store orderId and your refId — needed for all later calls | 400 on missing required fields |
| 4. Order detail | Poll status, get driver info, tracking URL. | status, terminal statuses, canCancel | 404 if wrong orderId |
| 5. Events | Full history of what happened. | — | 404 if wrong orderId |

Additional operations: update with `PUT /v1/orders/:orderId` (only after `ACCEPTED` and just before `PICKED_UP`), cancel with `DELETE /v1/orders/:orderId`, verify events with `POST /v1/orders/verify` or `POST /v1/orders/verify/picked-up/:refId`, verify pickup code with `POST /v1/customers/verify-pickup-code`.

#### 6.2 Same-day Order Flow (v2)
1. POST /v1/customers/favorite-address
2. GET /v1/nxp/sizes/same-day
3. POST /v1/nxp/sdd/timelines
4. POST /v2/orders

| Step | Why | Data carried forward |
|---|---|---|
| 1. Create Favorite Address | The pickup location must be a saved Favorite Address. | Favorite Address id |
| 2. Get sizes | Available sizes for Same-day delivery. | Chosen size |
| 3. Get SDD timelines | Available same-day time slots. Send sender_terminal_id as fav_id:<id>. | Chosen slot id |
| 4. Create order | POST /v2/orders with deliveryCategory: "same-day", refId, timeSlot.id, and packages (all required for Same-day). | Store orderId |

Track afterwards with `GET /v2/orders/:orderId` or `GET /v2/orders/references/:refId`.

#### 6.3 NXP Order Flow

:::note
Pre-action: before creating an NXP order, save the pickup terminal as a Favorite Address (POST /v1/customers/favorite-address). Use the saved Favorite Address ID as the pickup terminal ID in all NXP calls.
:::

1. GET /v1/nxp/sizes/:type
2. POST /v1/nxp/timelines
3. GET /v1/nxp/bundles (optional)
4. POST /v1/nxp/orders/invoice
5. POST /v1/nxp/orders
6. POST /v1/nxp/orders/:id/transactions

| Step | Why | Data carried forward |
|---|---|---|
| 1. Sizes | Get available parcel sizes. | Chosen size (type/value) |
| 2. Timelines | Get pickup/delivery time slots. | timeline_id |
| 3. Bundles (optional) | If you already have an order in ON_GOING status, list bundles to pick the pickup date of your new order based on the existing ON_GOING bundle. Requires bundle_status and sender_terminal_id query params. | Pickup date decision |
| 4. Invoice | Price quote. | token → invoice_token |
| 5. Create order | Create with invoice_token + timeline_id. Send unique X-Idempotency-Key header so retries never create duplicate orders. | Order id |
| 6. Pay | Pay from your Snapp Box wallet. Check balance first with GET /v1/wallets. | paid flag |

Afterwards: search orders (`GET /v1/nxp/orders`), parcels (`GET /v1/nxp/parcels`), labels (`GET /v1/nxp/parcels/labels`, returns HTML), cancel parcel (`DELETE /v1/nxp/parcels/:id`), list transactions (`GET /v1/nxp/transactions`).

### 7. Endpoint-by-Endpoint Implementation Checklist
| Endpoint | Method | Purpose | Request DTO | Response DTO | Required? | Important Validation |
|---|---|---|---|---|---|---|
| /v1/oauth2/token | POST | Get access token | TokenRequest | TokenResponse | Required (all flows) | client_id + client_secret required |
| /v1/orders/delivery-categories | GET | List delivery categories | — | DeliveryCategoriesResponse | Classic flow step 1 | ALL query params mandatory (latitude, longitude) |
| /v1/pricing | POST | Price quote (v1) | OrderCreateRequest-like | PricingResponse | Recommended (Classic) | Same required fields as create |
| /v1/orders | POST | Create Classic order | OrderCreateRequest v1 | Order detail (201) | Classic flow step 3 | refId, terminals, deliveryCategory, paymentType, city required |
| /v1/orders/:orderId | GET | Order detail | — | OrderDetail v1 | Required (tracking) | — |
| /v1/orders/references/:refId | GET | Order detail by refId | — | OrderDetail v1 | Optional | — |
| /v1/orders/list | GET | List orders | — | OrderListResponse | Optional | — |
| /v1/orders/:orderId/current-location | GET | Live driver location | — | OrderLocationResponse | Optional | — |
| /v1/orders/:orderId/events | GET | Event history | — | EventResponse[] | Optional | — |
| /v1/orders/:orderId | PUT | Update order | Update body | 204 no body | Optional | Only after ACCEPTED, before PICKED_UP |
| /v1/orders/:orderId | DELETE | Cancel order | — | 204 no body | Optional | — |
| /v1/orders/verify | POST | Verify order event | OrderVerifyRequest | 204 no body | Optional | verificationType enum: ARRIVED_AT_PICKUP, PICKED_UP, READY_FOR_PICKUP |
| /v1/orders/verify/picked-up/:refId | POST | Verify picked-up by refId | — | 204 no body | Optional | — |
| /v1/customers/verify-pickup-code | POST | Verify pickup code | VerifyPickupCodeRequest | 200 | Optional | pickupCode, terminalId, customerRefId required; orderId required if no customerRefId |
| /v1/orders/number-masking | POST | Mask phone numbers | NumberMaskingRequest | 204 no body | Optional | orderId + terminalId required |
| /v2/orders | POST | Create order (Classic or Same-day) | OrderCreateRequest v2 | CreateOrderResponse v2 (201) | v2 / Same-day flow | terminals, deliveryCategory, paymentType, city required; refId, timeSlot.id, packages required for Same-day |
| /v2/orders/:orderId | GET | Order detail v2 | — | OrderDetail v2 | v2 flow | — |
| /v2/orders/references/:refId | GET | Order detail by refId | — | OrderDetail v2 | Optional | — |
| /v1/customers/favorite-address | POST | Save favorite address | FavoriteAddress | FavoriteAddress (201) | NXP + Same-day pre-action | name, contactName, contactPhoneNumber, latitude, longitude, address required |
| /v1/customers/favorite-address | GET | List favorite addresses | — | FavoriteAddress[] | Optional | — |
| /v1/nxp/sizes/:delivery_type | GET | Parcel sizes | — | ParcelSize[] | NXP step 1 / Same-day step 2 | delivery_type: standard or same-day |
| /v1/nxp/timelines | POST | Search time slots | SearchTimelinesRequest | TimelineResponse[] | NXP step 2 | parcels required; receiver_terminal OR receiver_terminal_id |
| /v1/nxp/sdd/timelines | POST | Same-day time slots | SddTimelineRequest | SddTimelineResponse[] | Same-day step 3 | parcels + sender_terminal_id (fav_id:<id>) required |
| /v1/nxp/bundles | GET | List bundles | — | BundleResponse[] | Optional (NXP step 3) | bundle_status + sender_terminal_id query params required |
| /v1/nxp/orders/invoice | POST | Price quote | OrderInvoiceRequest | OrderInvoiceResponse | NXP step 4 | parcels required |
| /v1/nxp/orders | POST | Create NXP order | CreateOrderReq | CreateSenderOrderResponse (201) | NXP step 5 | invoice_token + parcels required; X-Idempotency-Key header |
| /v1/nxp/orders/:orderId/transactions | POST | Pay order | — | PaymentResponse | NXP step 6 | — |
| /v1/nxp/orders | GET | Search orders | — | FindOrdersResponse[] | Optional | — |
| /v1/nxp/parcels | GET | Search parcels | — | FindParcelsResponse[] | Optional | — |
| /v1/nxp/parcels/labels | GET | Parcel labels (HTML) | — | HTML page | Optional | Response is text/html, not JSON |
| /v1/nxp/parcels/:id | DELETE | Cancel parcel | ParcelCancelRequest | 204 no body | Optional | reason_id required |
| /v1/nxp/transactions | GET | List transactions | — | TransactionResponse[] | Optional | — |
| /v1/wallets | GET | Wallet balance | — | BalanceResponse | Recommended before NXP payment | — |
| /v1/insurances | GET | Insurance options | — | GetInsuranceResponse | Optional | — |

### 8. Client-Side Validation
Validate before sending. Failing fast locally avoids a wasted round-trip and a 400.

| Check | Rule |
|---|---|
| Required fields | Every field marked Required in section 5 must be present and non-empty. |
| Phone numbers | 11 digits only, e.g. 09121112233. Applies to phoneNumber, contactPhoneNumber, cellphone. |
| Enums | deliveryCategory must come from the delivery-categories call; verificationType must be ARRIVED_AT_PICKUP, PICKED_UP, or READY_FOR_PICKUP; Terminal type is pickup or drop. |
| Either/or fields | NXP parcels: exactly one of receiver_terminal or receiver_terminal_id. |
| Conditional requirements | Same-day orders: refId, timeSlot.id, and packages all required. VerifyPickupCode: orderId required when customerRefId is absent. |
| Formats | sender_terminal_id must be fav_id:<id>. Date/time fields ISO-8601 with Asia/Tehran offset. NXP Size value is a size ID for category type, or "L,W,H" for dimension type. |
| Numeric constraints | NXP city_id > 0; parcel value >= 0; reference 1–50 chars. |

### 9. Idempotency & Safe Retries of Creates

- Orders v1/v2: send a **unique `refId` per business order**. Retrying a create with the same `refId` is safe.

- NXP create: send a unique `X-Idempotency-Key` header so retries never create duplicate orders.

- Never invent a new `refId` / idempotency key on retry — reuse the original or you will create duplicates.

### 10. Error Handling
Every failed call returns the same shape: `key`, `message`, `statusCode` (see [Errors & Conventions](#errors)). Map them to your domain errors:

| Status | Meaning | Client action |
|---|---|---|
| 400 | Missing/invalid fields | Log the key, fix the request, do not retry unchanged. |
| 401 | Missing/expired token | Request a new token from /v1/oauth2/token, retry the call once with the new token. |
| 404 | Wrong ID in path | Fail; check stored orderId/refId. |
| 500 | Server/upstream failure | Retry with backoff (section 11); escalate to SnappBox support if persistent. |
| Network/timeout | No response received | Retry reads freely. Retry creates only with the original refId / X-Idempotency-Key (section 9). |

### 11. Retry Strategy
No retry policy is documented by the API — the following is a recommended client-side default:

- **Reads** (GET): retry on timeout, 500, and 429 with exponential backoff (e.g. 1s, 2s, 4s, max 3–5 attempts).

- **Creates** (POST orders): retry only with the original idempotency key (`refId` / `X-Idempotency-Key`) — otherwise a retry after a timeout may double-create.

- **401**: refresh token once, then retry once. A second 401 means credentials are wrong — fail loudly.

- **400**: never retry without changing the request.

### 12. Token Lifecycle

- Fetch a token from `POST /v1/oauth2/token` before the first protected call.

- Cache it; reuse until it expires (`expires_in` seconds).

- Proactively refresh shortly before expiry, or reactively on the first 401.

- Serialize refreshes: concurrent 401s should trigger one refresh, not many.

### 13. Testing Checklist
| Scenario | Expected result |
|---|---|
| Create order with a missing required field | Local validation rejects; no HTTP call made |
| Create order, then retry with same refId | No duplicate order created |
| Call with expired token | 401 → token refresh → retry succeeds |
| GET detail with unknown orderId | 404 mapped to your not-found error |
| Response containing unknown extra fields | Deserializer ignores them, no crash |
| Nullable fields returned as null | Deserializer tolerates nulls |
| Same-day create without timeSlot.id | Local validation rejects |
| NXP create retried with same X-Idempotency-Key | No duplicate order |

### 14. Documentation Gaps
Not documented by the API — decide client-side or confirm with SnappBox support:

- **Base URL** — obtain from SnappBox Support; never hardcode from docs.

- **HTTP timeout values** — pick client-side defaults (e.g. 10s connect / 30s read).

- **Retry policy / rate limits** — no documented limits; use the backoff in section 11.

- **Environments** — staging vs. production URLs not documented; ask support.

### 15. Logging & Monitoring

- Log every call with: endpoint, HTTP status, latency, `orderId`/`refId` when known, and a correlation ID per business operation.

- On errors, log the error `key` — it is the stable machine-readable identifier; `message` may change.

- Never log `client_secret`, access tokens, or full Authorization headers.

- Alert on: repeated 401s (credential problem), sustained 5xx (gateway problem), create retries that exhaust backoff.

- Track flow-completion metrics per flow in section 6: quote → create → paid/delivered rates surface integration breakage early.

