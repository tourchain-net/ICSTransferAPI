# ICS Transfer API V2 - Detailed Usage Guide

> **Base URL:** `https://tourchain.icstravelgroup.com/tourchain/api/v2/IcsTransfer`
> **Content-Type:** `application/json`
> **Authentication:** Bearer JWT Token (required for all endpoints)
> **Relationship to V1:** V2 is a completely independent endpoint from `api/IcsTransfer` (see [ICS_TRANSFER_API_GUIDE.md](ICS_TRANSFER_API_GUIDE.md)). It replaces the separate `arrival` / `departure` objects with a single unified `legs[]` array. A booking should be integrated through **one** version only — do not mix V1 and V2 calls for the same booking `number`.

---

## 1. Authentication

Identical to V1 — all endpoints require a **Bearer JWT Token** in the request header.

### Required Headers

| Header          | Value                            | Description                  |
|-----------------|----------------------------------|------------------------------|
| `Authorization` | `Bearer <jwt_token>`             | A valid JWT token            |
| `Content-Type`  | `application/json`               | Request content type         |

### Example Header

```http
Authorization: Bearer <jwt_token>
Content-Type: application/json
```

### Authentication Errors

| HTTP Status | Description                                      |
|-------------|--------------------------------------------------|
| `401`       | Token is invalid, expired, or missing            |

---

## 2. API: Create or Update Transfer Booking

### `POST /api/v2/IcsTransfer/webhook/booking-complete`

Creates a new or updates an existing ICS transfer booking, using the unified `legs[]` payload shape.

### Core differences vs. V1

- `transfer_information.arrival` / `transfer_information.departure` are **replaced** by `transfer_information.legs[]` — a single array.
- Each leg has a permanent `id` (a stable string, typically a UUID) — this is the **matching key** used on updates.
- **Every update must carry the complete set of legs** for the booking (full authoritative state, not a diff).
- A leg that existed on a previous update but is **absent** from the current update is automatically treated as **cancelled** — its `status` becomes `"cancelled"` or `"cancelled with charge"` depending on the configurable cancel time-limit rule (see Update Semantics), it is archived to `transferInformation.cancelledLegs[]`, and its downstream service/vehicle records are deactivated.
- Legs do **not** carry a `status` field in the request. Every active leg is stored with `status: "confirmed"`; cancellation status is derived by the system (absence from `legs[]` or the whole-order cancel endpoint).
- Leg count is variable: one leg, a matched pair, or multiple legs per direction (e.g. 2 arrival cars + 2 departure cars). **Do not assume 1:1 pairing between arrival and departure.**
- There is no top-level `transfer_information.type` or `transfer_information.notes` field in V2 — every leg is independently required, and notes are per-leg.

### 2.1 Request Body (single arrival + single departure)

```json
{
  "number": "B2024-VN-9988",
  "stopFollowUpButton": true,
  "customer_email": "nguyen.van.a@gmail.com",
  "customer_given_name": "Van A",
  "customer_surname": "Nguyen",
  "customer_phone": "+84912345678",
  "transfer_information": {
    "checkInDate": "2024-12-20",
    "checkOutDate": "2024-12-27",
    "fullname": "Nguyen Van A",
    "countryCodePhone": "+84",
    "phone": "0912345678",
    "luggage": 3,
    "oversizeLuggage": 0,
    "babyCarSeat": 1,
    "boosterSeat": 1,
    "pickUpDescription": "Hotel lobby, ground floor",
    "dropOffDescription": "International Terminal T1",
    "legs": [
      {
        "id": "11111111-2222-3333-4444-555555555555",
        "type": "arrival",
        "date": "2024-12-20",
        "time": "10:00",
        "flightNumber": "VN7201",
        "channelFareType": "DAD-AR-Z1SED",
        "adults": 2,
        "children": 1,
        "voucherCode": "7SNWFY9S",
        "pickUpDescription": "Danang International Airport (DAD)",
        "dropOffDescription": "InterContinental Danang Sun Peninsula",
        "notes": "Guest has an 18-month-old baby"
      },
      {
        "id": "66666666-7777-8888-9999-000000000000",
        "type": "departure",
        "date": "2024-12-27",
        "time": "08:30",
        "flightNumber": "VN7202",
        "channelFareType": "DAD-DE-Z1SED",
        "adults": 2,
        "children": 1,
        "voucherCode": "CFQSAPP3",
        "pickUpDescription": "InterContinental Danang Sun Peninsula",
        "dropOffDescription": "Danang International Airport (DAD)",
        "notes": ""
      }
    ]
  },
  "accommodation_items": [
    {
      "id": "acc-danang-2024",
      "reservation": {
        "check_in": "2024-12-20",
        "check_out": "2024-12-27",
        "hotel_info": {
          "id": "hotel-intercontinental-dn",
          "name": "InterContinental Danang Sun Peninsula",
          "geo_data": {
            "country": "Vietnam",
            "administrative_area_level_1": "Da Nang",
            "place_id": "ChIJrTLr-GyuEmsRBfy61i59si0"
          }
        }
      }
    }
  ]
}
```

---

### 2.2 Request Parameters

#### Root fields

| Field                | Type       | Required | Description                                                  |
|----------------------|------------|----------|----------------------------------------------------------------|
| `number`             | `string`   | ✅ Yes   | Booking number. e.g. `"B2024-VN-9988"`. Matches the whole order. |
| `stopFollowUpButton` | `boolean`  | ✅ Yes   | **Must be `true`**. Confirms the booking is complete            |
| `customer_email`     | `string`   | ✅ Yes   | Valid customer email address                                    |
| `customer_given_name`| `string`   | ✅ Yes   | Customer's first name                                            |
| `customer_surname`   | `string`   | ✅ Yes   | Customer's last name                                             |
| `customer_phone`     | `string`   | ✅ Yes   | Customer's phone number                                          |
| `transfer_information`| `object`  | ✅ Yes   | Detailed transfer information. See table below                  |
| `accommodation_items`| `array`    | ✅ Yes   | List of accommodation items. At least 1 item required            |

---

#### `transfer_information` object

| Field                | Type      | Required | Description                                                                      |
|----------------------|-----------|----------|------------------------------------------------------------------------------------|
| `checkInDate`        | `string`  | ❌ No    | Check-in date. Accepts `"YYYY-MM-DD"` or ISO datetime                              |
| `checkOutDate`       | `string`  | ❌ No    | Check-out date. Accepts `"YYYY-MM-DD"` or ISO datetime                             |
| `fullname`           | `string`  | ✅ Yes   | Full name of the lead passenger                                                    |
| `countryCodePhone`   | `string`  | ❌ No    | Country phone code. Accepts `"+84"` or `"84"`                                     |
| `phone`              | `string`  | ❌ No    | Phone number (without country code)                                                |
| `legs`               | `array`   | ✅ Yes   | **The complete set of transfer legs for this booking.** Must contain at least 1 item. See `legs[]` table below |
| `luggage`            | `integer` | ❌ No (default: `0`)   | Number of luggage pieces. Must be >= 0                              |
| `oversizeLuggage`    | `integer` | ❌ No (default: `0`)   | Oversize luggage. Accepts only `0` or `1`                           |
| `babyCarSeat`        | `integer` | ❌ No (default: `0`)   | Baby car seat required. Accepts only `0` or `1`                      |
| `boosterSeat`        | `integer` | ❌ No (default: `0`)   | Booster seat count. Accepts any non-negative integer                 |
| `pickUpDescription`  | `string`  | ❌ No    | Booking-level pick-up point. Fallback when a leg has no `pickUpDescription` |
| `dropOffDescription` | `string`  | ❌ No    | Booking-level drop-off point. Fallback when a leg has no `dropOffDescription` |

> Note: there is no `type` field. Whether the booking is arrival-only, departure-only, or both is now implied entirely by which leg `type` values appear in `legs[]`.

---

#### `legs[]` (Transfer Leg) object

| Field          | Type      | Required               | Description                                               |
|----------------|-----------|-------------------------|-------------------------------------------------------------|
| `id`           | `string`  | ✅ Yes                  | **Permanent, stable identifier for this leg** (recommend a UUID). This is the matching key on updates — reuse the same `id` for the same physical car/leg across calls. It is an opaque string; it is not required to be a formatted GUID. |
| `type`         | `string`  | ✅ Yes                  | Leg direction: `"arrival"` or `"departure"` (case-insensitive)  |
| `date`         | `string`  | ❌ No                   | Flight date. Accepts `"YYYY-MM-DD"` or ISO datetime. Can be empty, `null`, or `N/A` |
| `time`         | `string`  | ❌ No                   | Flight time. Accepts `"HH:mm"`, empty, `null`, or `N/A`         |
| `flightNumber` | `string`  | ❌ No                   | IATA flight number or empty, `null`, `N/A`                      |
| `channelFareType` | `string` | ❌ No                  | Transfer fare/service code used for downstream service sync. Can be empty, `null`, or `N/A`. Legs without a `channelFareType` are accepted but no downstream service is synced for them |
| `adults`       | `integer` | ✅ Yes                  | Number of adults for this leg. Must be >= 1                     |
| `children`     | `integer` | ❌ No (default: `0`)    | Number of children for this leg. Must be >= 0                   |
| `voucherCode`  | `string`  | ❌ No                   | LE voucher code for **this specific leg** (distinct from the root `number`). Reference only — not the matching key. Persisted as `legVoucherCode` on the downstream vehicle |
| `pickUpDescription` | `string` | ❌ No                | Pick-up point for this leg. Falls back to the top-level `transfer_information.pickUpDescription` if omitted |
| `dropOffDescription` | `string` | ❌ No               | Drop-off point for this leg. Falls back to the top-level `transfer_information.dropOffDescription` if omitted |
| `notes`        | `string`  | ❌ No                   | Per-leg special requests. Replaces the old top-level `transfer_information.notes` |

---

#### `accommodation_items[]`, `reservation`, `hotel_info`, `geo_data`

Identical to V1 — see the [V1 guide](ICS_TRANSFER_API_GUIDE.md#accommodation_items-array) for field tables. Unchanged in V2.

---

### 2.3 Validation Rules

| Field                                  | Rule                                                                       |
|-----------------------------------------|-----------------------------------------------------------------------------|
| `number`                                | Must not be empty                                                           |
| `stopFollowUpButton`                    | Must be `true`                                                              |
| `customer_email`                        | Must not be empty and must be a valid email address                         |
| `customer_given_name`                   | Must not be empty                                                           |
| `customer_surname`                      | Must not be empty                                                           |
| `customer_phone`                        | Must not be empty                                                           |
| `transfer_information`                  | Must not be null                                                            |
| `transfer_information.fullname`         | Must not be empty                                                           |
| `transfer_information.legs`             | Must not be empty — at least 1 leg is required                             |
| `transfer_information.legs`             | Every leg `id` must be unique within the payload (case-insensitive)         |
| `legs[].id`                             | Must not be empty                                                           |
| `legs[].type`                           | Must be `"arrival"` or `"departure"` (case-insensitive)                     |
| `legs[].date`                           | Optional. If provided, must be `YYYY-MM-DD` or a valid ISO datetime (or `N/A`) |
| `legs[].adults`                         | Must be >= 1                                                                |
| `legs[].children`                       | Must be >= 0                                                                |
| `luggage`                               | Must be >= 0                                                                |
| `oversizeLuggage`                       | Must be `0` or `1` only                                                    |
| `babyCarSeat`                           | Must be `0` or `1` only                                                    |
| `boosterSeat`                           | Must be >= 0                                                                |
| `accommodation_items`                   | Must not be empty. At least 1 item required                                |
| `accommodation_items[].id`              | Optional. Empty, `null`, and `N/A` are accepted                            |
| `reservation.check_in` / `check_out`    | Must not be empty                                                          |
| `hotel_info.id`                         | Optional. Empty, `null`, and `N/A` are accepted                            |
| `hotel_info.name`                       | Must not be empty                                                          |

---

### 2.4 Success Response

**HTTP Status: `200 OK`**

```json
{
  "status": 200,
  "message": "Booking accepted for processing"
}
```

The booking is validated synchronously and then queued for background processing (same asynchronous processing model as V1) — a `200` means the payload was accepted, not that every downstream service/vehicle sync has completed yet.

---

### 2.5 Error Response (Validation - 422)

**HTTP Status: `422 Unprocessable Entity`**

```json
{
  "status": 422,
  "error": 1,
  "messages": {
    "type": "error",
    "message": {
      "transfer_information.legs": "Each leg id must be unique within legs",
      "transfer_information.legs[0].adults": "At least 1 adult required for each leg",
      "transfer_information.legs[1].type": "Leg type must be 'arrival' or 'departure'"
    }
  }
}
```

| Field                | Type             | Description                                                         |
|----------------------|------------------|-----------------------------------------------------------------------|
| `status`             | `integer`        | HTTP status code                                                     |
| `error`              | `integer`        | Always `1` when an error occurs                                      |
| `messages.type`      | `string`         | Error type, always `"error"`                                         |
| `messages.message`   | `string\|object` | A plain error string, or an object with field names as keys           |

---

### 2.6 cURL Example

```bash
curl -X POST https://tourchain.icstravelgroup.com/tourchain/api/v2/IcsTransfer/webhook/booking-complete \
  -H "Authorization: Bearer <jwt_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "number": "B2024-VN-9988",
    "stopFollowUpButton": true,
    "customer_email": "nguyen.van.a@gmail.com",
    "customer_given_name": "Van A",
    "customer_surname": "Nguyen",
    "customer_phone": "+84912345678",
    "transfer_information": {
      "fullname": "Nguyen Van A",
      "luggage": 2,
      "oversizeLuggage": 0,
      "babyCarSeat": 0,
      "boosterSeat": 0,
      "legs": [
        {
          "id": "11111111-2222-3333-4444-555555555555",
          "type": "arrival",
          "date": "2024-12-20",
          "time": "10:00",
          "flightNumber": "VN7201",
          "channelFareType": "DAD-AR-Z1SED",
          "adults": 2,
          "children": 0
        }
      ]
    },
    "accommodation_items": [
      {
        "id": "hotel-item-001",
        "reservation": {
          "check_in": "2024-12-20",
          "check_out": "2024-12-27",
          "hotel_info": {
            "id": "hotel-intercontinental-dn",
            "name": "InterContinental Danang Sun Peninsula"
          }
        }
      }
    ]
  }'
```

---

## 3. Update Semantics (V2-specific)

Unlike V1, where the arrival/departure objects were merged field-by-field, V2 treats every update's `legs[]` as the **complete, authoritative state** of the booking's legs. On each `booking-complete` call:

1. **Matched by `id`** — a leg in the payload whose `id` matches a leg already stored on the booking is updated in place. Any downstream service linkage already resolved for that leg (`serviceAdded`, matched supplier service, assigned vehicle, user-entered `pickUpTime`) is preserved across the update — you do not need to re-send those derived fields.
2. **New `id`** — a leg whose `id` was never seen before is treated as a brand-new leg: a new downstream service/vehicle is created for it (if it has a `channelFareType`).
3. **Missing `id`** — a leg that existed on a previous call but is **not present** in the current call's `legs[]` is automatically cancelled:
   - Its `status` is set by the **cancel time-limit rule** (default 24 hours, configurable via the admin cancel-policy config — see `ICS_CANCEL_POLICY_CONFIG_FE_GUIDE.md`): if the leg's `date` + `time` (in the booking country's local time, per-country UTC offset from config, default +8) is **more than the limit** away from the current time, the status is `"cancelled"`; if it is **within the limit** (or already in the past), the status is `"cancelled with charge"`. A leg with a missing/unparseable `date` gets `"cancelled"`; a missing `time` is treated as `00:00`.
   - The leg is moved into `transferInformation.cancelledLegs[]` with a `cancelledDate` and a `cancelPolicy` snapshot of the values used for the decision: `{ "cancelLimitHours": 24, "utcOffset": 7, "nation": "Vietnam" }`.
   - Its downstream service/vehicle records are deactivated.
   - This happens automatically — you do not call a separate "remove leg" endpoint. To cancel a single car/leg, simply resend the booking without that leg's `id`.

**Leg count is not fixed.** A booking may have 1 leg, a matched arrival/departure pair, or multiple legs per direction (e.g. two arrival cars and one departure car, or vice versa). Do not assume any pairing between arrival and departure legs.

---

## 4. API: Cancel Transfer Booking (whole order)

### `POST /api/v2/IcsTransfer/webhook/cancel`

Cancels an entire ICS transfer order by booking number and customer email. Use this to cancel the whole booking — for cancelling a single leg/car, simply resend `booking-complete` without that leg (see Update Semantics above).

> **Why `customer_email` is required:** the same voucher number may exist on multiple leads (different customers). Providing the customer email ensures the correct customer's booking is cancelled. If the email does not match any lead holding that voucher, the API returns `404` with a list of emails found on the matching leads.

### 4.1 Request Body (JSON)

```json
{
  "number": "TC-DPS-000011",
  "customer_email": "test10@tourchain.com"
}
```

| Field            | Type     | Required | Description                                                      |
|------------------|----------|----------|------------------------------------------------------------------|
| `number`         | `string` | ✅ Yes   | The booking/voucher number to cancel. e.g. `"TC-DPS-000011"`     |
| `customer_email` | `string` | ✅ Yes   | Customer email to disambiguate when the same voucher number exists across multiple customers (case-insensitive exact match). |

### 4.2 Processing Flow

1. Look up the booking by `number`.
2. Match by `customer_email` (case-insensitive). If no match is found, return `404` with a message listing the emails found on candidate bookings.
3. Mark the booking as cancelled.
4. Apply the **cancel time-limit rule** to every leg (same rule as booking-complete — cancel-policy config, default 24h): a leg further than the limit → `"cancelled"`, a leg within the limit or in the past → `"cancelled with charge"`. Each leg also gets `cancelledDate` and a `cancelPolicy` snapshot: `{ "cancelLimitHours": 24, "utcOffset": 8, "nation": "Indonesia" }`.
5. Deactivate all downstream services and vehicles linked to the cancelled legs. Each vehicle's status is set to match its leg's resolved cancellation status (`"cancelled"` or `"cancelled with charge"`).

### 4.3 Success Response

**HTTP Status: `200 OK`**

```json
{
  "status": 200,
  "message": "Booking cancelled successfully"
}
```

### 4.4 Error Responses

**HTTP Status: `422 Unprocessable Entity`** — when `number` or `customer_email` is missing/invalid:

```json
{
  "status": 422,
  "error": 1,
  "messages": {
    "type": "error",
    "message": "Booking number is required"
  }
}
```

**HTTP Status: `404 Not Found`** — when `number` does not exist:

```json
{
  "status": 404,
  "error": 1,
  "messages": {
    "type": "error",
    "message": "Booking 'TC-DPS-000011' not found"
  }
}
```

**HTTP Status: `404 Not Found`** — when `number` exists on lead(s) but `customer_email` matches none:

```json
{
  "status": 404,
  "error": 1,
  "messages": {
    "type": "error",
    "message": "Booking 'TC-DPS-000011' found on 2 lead(s) with email(s) [alice@test.com, bob@test.com] — none match 'wrong@test.com'. Provide the correct customer_email to cancel."
  }
}
```

### 4.5 cURL Example

```bash
curl -X POST https://tourchain.icstravelgroup.com/tourchain/api/v2/IcsTransfer/webhook/cancel \
  -H "Authorization: Bearer <jwt_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "number": "TC-DPS-000011",
    "customer_email": "test10@tourchain.com"
  }'
```

---

## 5. HTTP Status Codes Summary

| Status Code | Scenario                                             |
|-------------|--------------------------------------------------------|
| `200`       | Success                                                |
| `401`       | Token is invalid, expired, or missing                  |
| `404`       | Booking not found (number doesn't exist, or number exists but customer_email doesn't match any lead) |
| `422`       | Input validation failed (missing number, invalid email, etc.) |

---

## 6. Full Example: arrival-and-departure (1 car each way)

```bash
curl -X POST https://tourchain.icstravelgroup.com/tourchain/api/v2/IcsTransfer/webhook/booking-complete \
  -H "Authorization: Bearer <jwt_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "number": "B2024-VN-9988",
    "stopFollowUpButton": true,
    "customer_email": "nguyen.van.a@gmail.com",
    "customer_given_name": "Van A",
    "customer_surname": "Nguyen",
    "customer_phone": "+84912345678",
    "transfer_information": {
      "checkInDate": "2024-12-20",
      "checkOutDate": "2024-12-27",
      "fullname": "Nguyen Van A",
      "countryCodePhone": "+84",
      "phone": "0912345678",
      "luggage": 3,
      "oversizeLuggage": 0,
      "babyCarSeat": 1,
      "boosterSeat": 1,
      "pickUpDescription": "Hotel lobby, ground floor",
      "dropOffDescription": "International Terminal T1",
      "legs": [
        {
          "id": "11111111-2222-3333-4444-555555555555",
          "type": "arrival",
          "date": "2024-12-20",
          "time": "10:00",
          "flightNumber": "VN7201",
          "channelFareType": "DAD-AR-Z1SED",
          "adults": 2,
          "children": 1,
          "notes": "Guest has an 18-month-old baby"
        },
        {
          "id": "66666666-7777-8888-9999-000000000000",
          "type": "departure",
          "date": "2024-12-27",
          "time": "08:30",
          "flightNumber": "VN7202",
          "channelFareType": "DAD-DE-Z1SED",
          "adults": 2,
          "children": 1,
          "notes": ""
        }
      ]
    },
    "accommodation_items": [
      {
        "id": "acc-danang-2024",
        "reservation": {
          "check_in": "2024-12-20",
          "check_out": "2024-12-27",
          "hotel_info": {
            "id": "hotel-intercontinental-dn",
            "name": "InterContinental Danang Sun Peninsula",
            "geo_data": {
              "country": "Vietnam",
              "administrative_area_level_1": "Da Nang",
              "place_id": "ChIJrTLr-GyuEmsRBfy61i59si0"
            }
          }
        }
      }
    ]
  }'
```

**Response:**

```json
{
  "status": 200,
  "message": "Booking accepted for processing"
}
```

---

## 7. Example: Multiple Cars in the Same Direction

Leg count is not fixed and there is no 1:1 pairing requirement — a booking can have several legs of the same `type`. Each car is its own leg with its own `id`; only legs with a `channelFareType` get a downstream service synced.

```json
{
  "number": "7F4DA7",
  "stopFollowUpButton": true,
  "customer_email": "group@example.com",
  "customer_given_name": "Group",
  "customer_surname": "Lead",
  "customer_phone": "61400000000",
  "transfer_information": {
    "checkInDate": "2026-09-04T16:20:00",
    "checkOutDate": "2026-09-10T11:00:00",
    "fullname": "Group Lead",
    "countryCodePhone": "61",
    "phone": "400000000",
    "luggage": 6,
    "oversizeLuggage": 0,
    "babyCarSeat": 0,
    "boosterSeat": 0,
    "pickUpDescription": "Bali Denpasar",
    "dropOffDescription": "Nusa Dua Hotel",
    "legs": [
      {
        "id": "aaaaaaaa-0000-0000-0000-000000000001",
        "type": "arrival",
        "date": "2026-09-04T16:20:00",
        "time": "16:20",
        "flightNumber": "NZ290",
        "adults": 3,
        "children": 0,
        "channelFareType": "DPS-AR-Z1SED",
        "pickUpDescription": "Bali Denpasar",
        "dropOffDescription": "Nusa Dua Hotel"
      },
      {
        "id": "aaaaaaaa-0000-0000-0000-000000000002",
        "type": "arrival",
        "date": "2026-09-04T16:20:00",
        "time": "16:20",
        "flightNumber": "NZ290",
        "adults": 3,
        "children": 0,
        "channelFareType": "DPS-AR-Z1SED",
        "pickUpDescription": "Bali Denpasar",
        "dropOffDescription": "Nusa Dua Hotel"
      },
      {
        "id": "bbbbbbbb-0000-0000-0000-000000000001",
        "type": "departure",
        "date": "2026-09-10T11:00:00",
        "time": "11:00",
        "flightNumber": "NZ291",
        "adults": 3,
        "children": 0,
        "channelFareType": "DPS-DE-Z1SED",
        "pickUpDescription": "Nusa Dua Hotel",
        "dropOffDescription": "Bali Denpasar"
      },
      {
        "id": "bbbbbbbb-0000-0000-0000-000000000002",
        "type": "departure",
        "date": "2026-09-10T11:00:00",
        "time": "11:00",
        "flightNumber": "NZ291",
        "adults": 3,
        "children": 0,
        "channelFareType": "DPS-DE-Z1SED",
        "pickUpDescription": "Nusa Dua Hotel",
        "dropOffDescription": "Bali Denpasar"
      }
    ]
  },
  "accommodation_items": [
    {
      "id": "cccccccc-0000-0000-0000-000000000001",
      "reservation": {
        "check_in": "2026-09-04T16:20:00",
        "check_out": "2026-09-10T11:00:00",
        "hotel_info": {
          "id": "HTL-EXAMPLE-002",
          "name": "Nusa Dua Hotel",
          "geo_data": {
            "administrative_area_level_1": "Bali",
            "country": "Indonesia",
            "place_id": "ChIJexample001"
          }
        }
      }
    }
  ]
}
```

A follow-up call that cancels only the second arrival car (`aaaaaaaa-0000-0000-0000-000000000002`) simply omits that leg from `legs[]` while resending the other three — no separate cancel call is needed for a single leg.

---

## 8. Cancel Booking Example

```bash
curl -X DELETE https://tourchain.icstravelgroup.com/tourchain/api/v2/IcsTransfer/webhook/cancel/B2024-VN-9988 \
  -H "Authorization: Bearer <jwt_token>"
```

**Response:**

```json
{
  "status": 200,
  "message": "Booking updated successfully"
}
```

---

## 9. Important Notes

- **`stopFollowUpButton` must always be `true`** — the server will reject the request if the value is `false`.
- There is no `transfer_information.type` field in V2. Every leg present in `legs[]` is validated and processed independently; the booking's overall direction mix is simply whatever `type` values appear across the legs.
- **`oversizeLuggage` and `babyCarSeat`** only accept integer values `0` or `1` (not boolean `true`/`false`). `boosterSeat` accepts any non-negative integer. Same as V1.
- **JWT Token** must be sent in the correct format: `Bearer <token>` (with a space between `Bearer` and the token).
- **Every `booking-complete` call must resend the full, current set of legs.** Do not send only the legs that changed — any leg id omitted from the payload is treated as cancelled.
- Leg `id` should be **stable and reused** across calls for the same physical leg/car — this is how the system tells "update this leg" apart from "add a new leg" and "cancel a leg by omission".
- For backward compatibility with older dashboard views and downstream sync, the system internally derives a legacy-shaped `arrival` / `departure` sub-document from the first leg of each direction. This is an internal detail — LE does not need to produce or consume it — but note that if a booking has multiple legs in the same direction, older dashboard views built for V1 will only display the first leg of that direction.
- V1 (`/api/IcsTransfer`) and V2 (`/api/v2/IcsTransfer`) are independent endpoints with independent processing pipelines. Choose one version per booking `number` and do not send the same booking through both.

### Vehicle Data Saved from ICS Transfer V2

When a booking is processed, the following fields from the request are persisted to each vehicle record in the downstream booking:

| Request Field                          | Vehicle Field          | Source                          |
|------------------------------------------|--------------------------|------------------------------------|
| `transfer_information.luggage`         | `luggage`              | `transferInformation`            |
| `transfer_information.oversizeLuggage` | `oversizeLuggage`      | `transferInformation`            |
| `transfer_information.babyCarSeat`     | `babyCarSeat`          | `transferInformation`            |
| `transfer_information.boosterSeat`     | `boosterSeat`          | `transferInformation`            |
| `legs[].adults`                        | `adultsCount`          | Leg document                     |
| `legs[].children`                      | `childrenCount`        | Leg document                     |
| `legs[].voucherCode`                   | `legVoucherCode`       | Leg document (per-leg voucher; root `number` still maps to `voucherCode`) |
| (resolved from leg services)           | `packageName`          | Resolved from leg services       |
| (resolved from leg services)           | `packageInternalName`  | Resolved from leg services       |
| (resolved from leg)                    | `flightInfo`           | Built from flight number + time  |
| (resolved from transfer)               | `hotelName`            | Resolved from accommodation items|
| `legs[].pickUpDescription` / `dropOffDescription` | `pickupPoint` / `dropOffPoint` | Per-leg route when present; otherwise the top-level `pickUpDescription`/`dropOffDescription` (swapped for departure legs) |
| `legs[].notes`                          | `note`                  | Leg document                     |

---

## 10. Recommended Test Scenarios

Use these scenarios to validate the V2 API integration end-to-end:

| Scenario | How to Test | Expected Result |
|----------|-------------|-----------------|
| **Create new booking** | Call `POST /booking-complete` with a new `number` and 2 legs (1 arrival, 1 departure). All leg `id`s are new. | HTTP `200`. Both legs stored with `status: "confirmed"`. Downstream services created for each leg with a `channelFareType`. |
| **Update a single leg** | Call `POST /booking-complete` with the same `number`. Keep the same leg `id`s but change one leg's `flightNumber`, `time`, or `notes`. | HTTP `200`. The updated leg is modified in-place. Other legs unchanged. Downstream service/vehicle links (e.g., `serviceAdded`, `pickUpTime`) preserved for the updated leg. |
| **Cancel a single car by omitting leg** | Call `POST /booking-complete` with the same `number`. Omit one leg's `id` from `legs[]` while resending the others unchanged. | HTTP `200`. The omitted leg's `status` becomes `"cancelled"` or `"cancelled with charge"` based on the cancel time-limit rule (default 24h). Leg is moved to `cancelledLegs[]` with `cancelledDate` and `cancelPolicy` snapshot. Corresponding vehicle is deactivated. |
| **Cancel entire booking order** | Call `DELETE /webhook/cancel/{number}` where `{number}` is the booking number. | HTTP `200`. All legs are cancelled with status derived from cancel time-limit rule. All vehicles deactivated. Message: `"Booking updated successfully"`. |
| **Booking number does not exist (cancel)** | Call `DELETE /webhook/cancel/{number}` with a booking number that does not exist. | HTTP `404` or appropriate error response indicating the booking was not found. |
| **Add a new leg to existing booking** | Call `POST /booking-complete` with the same `number`. Include all previous leg `id`s plus one new leg with a new unique `id`. | HTTP `200`. New leg stored with `status: "confirmed"`. If it has a `channelFareType`, a new downstream service is created. Previous legs updated if their fields changed, or left unchanged. |
| **Missing required field** | Call `POST /booking-complete` with missing `number`, `customer_email`, or `transfer_information` field. | HTTP `422`. Error message specifying the missing required field. |
| **Invalid leg data** | Call `POST /booking-complete` with duplicate leg `id`s, missing `adults` count, or invalid leg `type`. | HTTP `422`. Error message specifying the validation error. |

---

## 10.1 Test Payloads and cURL Examples

### Scenario 1: Create New Booking

**Payload:**
```json
{
  "number": "TEST-001-NEW",
  "stopFollowUpButton": true,
  "customer_email": "newcustomer@example.com",
  "customer_given_name": "John",
  "customer_surname": "Smith",
  "customer_phone": "+61412345678",
  "transfer_information": {
    "checkInDate": "2026-08-15",
    "checkOutDate": "2026-08-20",
    "fullname": "John Smith",
    "countryCodePhone": "61",
    "phone": "412345678",
    "luggage": 2,
    "oversizeLuggage": 0,
    "babyCarSeat": 0,
    "boosterSeat": 0,
    "pickUpDescription": "Sydney International Airport (SYD)",
    "dropOffDescription": "Hilton Sydney",
    "legs": [
      {
        "id": "aaaaaaaa-1111-1111-1111-111111111111",
        "type": "arrival",
        "date": "2026-08-15",
        "time": "14:30",
        "flightNumber": "QF001",
        "channelFareType": "SYD-AR-Z1SED",
        "adults": 2,
        "children": 0,
        "pickUpDescription": "Sydney International Airport (SYD)",
        "dropOffDescription": "Hilton Sydney",
        "notes": ""
      },
      {
        "id": "bbbbbbbb-2222-2222-2222-222222222222",
        "type": "departure",
        "date": "2026-08-20",
        "time": "11:00",
        "flightNumber": "QF002",
        "channelFareType": "SYD-DE-Z1SED",
        "adults": 2,
        "children": 0,
        "pickUpDescription": "Hilton Sydney",
        "dropOffDescription": "Sydney International Airport (SYD)",
        "notes": ""
      }
    ]
  },
  "accommodation_items": [
    {
      "id": "hotel-001",
      "reservation": {
        "check_in": "2026-08-15",
        "check_out": "2026-08-20",
        "hotel_info": {
          "id": "hilton-syd-001",
          "name": "Hilton Sydney",
          "geo_data": {
            "country": "Australia",
            "administrative_area_level_1": "New South Wales",
            "place_id": "ChIJrTLr-GyuEmsRBfy61i59si0"
          }
        }
      }
    }
  ]
}
```

**cURL:**
```bash
curl -X POST https://tourchain.icstravelgroup.com/tourchain/api/v2/IcsTransfer/webhook/booking-complete \
  -H "Authorization: Bearer <jwt_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "number": "TEST-001-NEW",
    "stopFollowUpButton": true,
    "customer_email": "newcustomer@example.com",
    "customer_given_name": "John",
    "customer_surname": "Smith",
    "customer_phone": "+61412345678",
    "transfer_information": {
      "checkInDate": "2026-08-15",
      "checkOutDate": "2026-08-20",
      "fullname": "John Smith",
      "countryCodePhone": "61",
      "phone": "412345678",
      "luggage": 2,
      "oversizeLuggage": 0,
      "babyCarSeat": 0,
      "boosterSeat": 0,
      "pickUpDescription": "Sydney International Airport (SYD)",
      "dropOffDescription": "Hilton Sydney",
      "legs": [
        {
          "id": "aaaaaaaa-1111-1111-1111-111111111111",
          "type": "arrival",
          "date": "2026-08-15",
          "time": "14:30",
          "flightNumber": "QF001",
          "channelFareType": "SYD-AR-Z1SED",
          "adults": 2,
          "children": 0,
          "pickUpDescription": "Sydney International Airport (SYD)",
          "dropOffDescription": "Hilton Sydney",
          "notes": ""
        },
        {
          "id": "bbbbbbbb-2222-2222-2222-222222222222",
          "type": "departure",
          "date": "2026-08-20",
          "time": "11:00",
          "flightNumber": "QF002",
          "channelFareType": "SYD-DE-Z1SED",
          "adults": 2,
          "children": 0,
          "pickUpDescription": "Hilton Sydney",
          "dropOffDescription": "Sydney International Airport (SYD)",
          "notes": ""
        }
      ]
    },
    "accommodation_items": [
      {
        "id": "hotel-001",
        "reservation": {
          "check_in": "2026-08-15",
          "check_out": "2026-08-20",
          "hotel_info": {
            "id": "hilton-syd-001",
            "name": "Hilton Sydney",
            "geo_data": {
              "country": "Australia",
              "administrative_area_level_1": "New South Wales",
              "place_id": "ChIJrTLr-GyuEmsRBfy61i59si0"
            }
          }
        }
      }
    ]
  }'
```

**Expected Response:**
```json
{
  "status": 200,
  "message": "Booking accepted for processing"
}
```

---

### Scenario 2: Update a Single Leg

Use the same `number` and leg `id`s as Scenario 1, but change the arrival leg's `flightNumber` and `time`.

**Payload:**
```json
{
  "number": "TEST-001-NEW",
  "stopFollowUpButton": true,
  "customer_email": "newcustomer@example.com",
  "customer_given_name": "John",
  "customer_surname": "Smith",
  "customer_phone": "+61412345678",
  "transfer_information": {
    "checkInDate": "2026-08-15",
    "checkOutDate": "2026-08-20",
    "fullname": "John Smith",
    "countryCodePhone": "61",
    "phone": "412345678",
    "luggage": 2,
    "oversizeLuggage": 0,
    "babyCarSeat": 0,
    "boosterSeat": 0,
    "pickUpDescription": "Sydney International Airport (SYD)",
    "dropOffDescription": "Hilton Sydney",
    "legs": [
      {
        "id": "aaaaaaaa-1111-1111-1111-111111111111",
        "type": "arrival",
        "date": "2026-08-15",
        "time": "16:45",
        "flightNumber": "QF101",
        "channelFareType": "SYD-AR-Z1SED",
        "adults": 2,
        "children": 0,
        "pickUpDescription": "Sydney International Airport (SYD)",
        "dropOffDescription": "Hilton Sydney",
        "notes": "Flight delayed, new ETA 16:45"
      },
      {
        "id": "bbbbbbbb-2222-2222-2222-222222222222",
        "type": "departure",
        "date": "2026-08-20",
        "time": "11:00",
        "flightNumber": "QF002",
        "channelFareType": "SYD-DE-Z1SED",
        "adults": 2,
        "children": 0,
        "pickUpDescription": "Hilton Sydney",
        "dropOffDescription": "Sydney International Airport (SYD)",
        "notes": ""
      }
    ]
  },
  "accommodation_items": [
    {
      "id": "hotel-001",
      "reservation": {
        "check_in": "2026-08-15",
        "check_out": "2026-08-20",
        "hotel_info": {
          "id": "hilton-syd-001",
          "name": "Hilton Sydney",
          "geo_data": {
            "country": "Australia",
            "administrative_area_level_1": "New South Wales",
            "place_id": "ChIJrTLr-GyuEmsRBfy61i59si0"
          }
        }
      }
    }
  ]
}
```

**cURL:**
```bash
curl -X POST https://tourchain.icstravelgroup.com/tourchain/api/v2/IcsTransfer/webhook/booking-complete \
  -H "Authorization: Bearer <jwt_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "number": "TEST-001-NEW",
    "stopFollowUpButton": true,
    "customer_email": "newcustomer@example.com",
    "customer_given_name": "John",
    "customer_surname": "Smith",
    "customer_phone": "+61412345678",
    "transfer_information": {
      "checkInDate": "2026-08-15",
      "checkOutDate": "2026-08-20",
      "fullname": "John Smith",
      "countryCodePhone": "61",
      "phone": "412345678",
      "luggage": 2,
      "oversizeLuggage": 0,
      "babyCarSeat": 0,
      "boosterSeat": 0,
      "pickUpDescription": "Sydney International Airport (SYD)",
      "dropOffDescription": "Hilton Sydney",
      "legs": [
        {
          "id": "aaaaaaaa-1111-1111-1111-111111111111",
          "type": "arrival",
          "date": "2026-08-15",
          "time": "16:45",
          "flightNumber": "QF101",
          "channelFareType": "SYD-AR-Z1SED",
          "adults": 2,
          "children": 0,
          "pickUpDescription": "Sydney International Airport (SYD)",
          "dropOffDescription": "Hilton Sydney",
          "notes": "Flight delayed, new ETA 16:45"
        },
        {
          "id": "bbbbbbbb-2222-2222-2222-222222222222",
          "type": "departure",
          "date": "2026-08-20",
          "time": "11:00",
          "flightNumber": "QF002",
          "channelFareType": "SYD-DE-Z1SED",
          "adults": 2,
          "children": 0,
          "pickUpDescription": "Hilton Sydney",
          "dropOffDescription": "Sydney International Airport (SYD)",
          "notes": ""
        }
      ]
    },
    "accommodation_items": [
      {
        "id": "hotel-001",
        "reservation": {
          "check_in": "2026-08-15",
          "check_out": "2026-08-20",
          "hotel_info": {
            "id": "hilton-syd-001",
            "name": "Hilton Sydney",
            "geo_data": {
              "country": "Australia",
              "administrative_area_level_1": "New South Wales",
              "place_id": "ChIJrTLr-GyuEmsRBfy61i59si0"
            }
          }
        }
      }
    ]
  }'
```

**Expected Response:**
```json
{
  "status": 200,
  "message": "Booking accepted for processing"
}
```

---

### Scenario 3: Cancel a Single Car by Omitting Leg

Use the same `number` and keep both leg `id`s, but remove the departure leg from the `legs[]` array.

**Payload (departure leg omitted):**
```json
{
  "number": "TEST-001-NEW",
  "stopFollowUpButton": true,
  "customer_email": "newcustomer@example.com",
  "customer_given_name": "John",
  "customer_surname": "Smith",
  "customer_phone": "+61412345678",
  "transfer_information": {
    "checkInDate": "2026-08-15",
    "checkOutDate": "2026-08-20",
    "fullname": "John Smith",
    "countryCodePhone": "61",
    "phone": "412345678",
    "luggage": 2,
    "oversizeLuggage": 0,
    "babyCarSeat": 0,
    "boosterSeat": 0,
    "pickUpDescription": "Sydney International Airport (SYD)",
    "dropOffDescription": "Hilton Sydney",
    "legs": [
      {
        "id": "aaaaaaaa-1111-1111-1111-111111111111",
        "type": "arrival",
        "date": "2026-08-15",
        "time": "16:45",
        "flightNumber": "QF101",
        "channelFareType": "SYD-AR-Z1SED",
        "adults": 2,
        "children": 0,
        "pickUpDescription": "Sydney International Airport (SYD)",
        "dropOffDescription": "Hilton Sydney",
        "notes": "Flight delayed, new ETA 16:45"
      }
    ]
  },
  "accommodation_items": [
    {
      "id": "hotel-001",
      "reservation": {
        "check_in": "2026-08-15",
        "check_out": "2026-08-20",
        "hotel_info": {
          "id": "hilton-syd-001",
          "name": "Hilton Sydney",
          "geo_data": {
            "country": "Australia",
            "administrative_area_level_1": "New South Wales",
            "place_id": "ChIJrTLr-GyuEmsRBfy61i59si0"
          }
        }
      }
    }
  ]
}
```

**cURL:**
```bash
curl -X POST https://tourchain.icstravelgroup.com/tourchain/api/v2/IcsTransfer/webhook/booking-complete \
  -H "Authorization: Bearer <jwt_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "number": "TEST-001-NEW",
    "stopFollowUpButton": true,
    "customer_email": "newcustomer@example.com",
    "customer_given_name": "John",
    "customer_surname": "Smith",
    "customer_phone": "+61412345678",
    "transfer_information": {
      "checkInDate": "2026-08-15",
      "checkOutDate": "2026-08-20",
      "fullname": "John Smith",
      "countryCodePhone": "61",
      "phone": "412345678",
      "luggage": 2,
      "oversizeLuggage": 0,
      "babyCarSeat": 0,
      "boosterSeat": 0,
      "pickUpDescription": "Sydney International Airport (SYD)",
      "dropOffDescription": "Hilton Sydney",
      "legs": [
        {
          "id": "aaaaaaaa-1111-1111-1111-111111111111",
          "type": "arrival",
          "date": "2026-08-15",
          "time": "16:45",
          "flightNumber": "QF101",
          "channelFareType": "SYD-AR-Z1SED",
          "adults": 2,
          "children": 0,
          "pickUpDescription": "Sydney International Airport (SYD)",
          "dropOffDescription": "Hilton Sydney",
          "notes": "Flight delayed, new ETA 16:45"
        }
      ]
    },
    "accommodation_items": [
      {
        "id": "hotel-001",
        "reservation": {
          "check_in": "2026-08-15",
          "check_out": "2026-08-20",
          "hotel_info": {
            "id": "hilton-syd-001",
            "name": "Hilton Sydney",
            "geo_data": {
              "country": "Australia",
              "administrative_area_level_1": "New South Wales",
              "place_id": "ChIJrTLr-GyuEmsRBfy61i59si0"
            }
          }
        }
      }
    ]
  }'
```

**Expected Response:**
```json
{
  "status": 200,
  "message": "Booking accepted for processing"
}
```

**Expected State Change:** The departure leg (`bbbbbbbb-2222-2222-2222-222222222222`) is automatically moved to `cancelledLegs[]` with status `"cancelled"` or `"cancelled with charge"` (based on cancel time-limit rule). The corresponding vehicle is deactivated.

---

### Scenario 4: Cancel Entire Booking Order

Cancel the whole booking using the `DELETE` endpoint.

**cURL:**
```bash
curl -X DELETE https://tourchain.icstravelgroup.com/tourchain/api/v2/IcsTransfer/webhook/cancel/TEST-001-NEW \
  -H "Authorization: Bearer <jwt_token>"
```

**Expected Response:**
```json
{
  "status": 200,
  "message": "Booking updated successfully"
}
```

**Expected State Change:** All legs (arrival and any remaining legs) are moved to `cancelledLegs[]` with status determined by cancel time-limit rule. All vehicles are deactivated.

---

### Scenario 5: Booking Number Does Not Exist (Cancel)

Attempt to cancel with a booking number that does not exist.

**cURL:**
```bash
curl -X DELETE https://tourchain.icstravelgroup.com/tourchain/api/v2/IcsTransfer/webhook/cancel/NONEXISTENT-BOOKING-123 \
  -H "Authorization: Bearer <jwt_token>"
```

**Expected Response (404):**
```json
{
  "status": 404,
  "error": 1,
  "messages": {
    "type": "error",
    "message": "Booking 'NONEXISTENT-BOOKING-123' not found"
  }
}
```

---

### Scenario 6: Add a New Leg to Existing Booking

Use the same `number` and existing leg `id`s, but add a brand-new leg with a unique `id`.

**Payload (with new departure leg added):**
```json
{
  "number": "TEST-001-NEW",
  "stopFollowUpButton": true,
  "customer_email": "newcustomer@example.com",
  "customer_given_name": "John",
  "customer_surname": "Smith",
  "customer_phone": "+61412345678",
  "transfer_information": {
    "checkInDate": "2026-08-15",
    "checkOutDate": "2026-08-20",
    "fullname": "John Smith",
    "countryCodePhone": "61",
    "phone": "412345678",
    "luggage": 2,
    "oversizeLuggage": 0,
    "babyCarSeat": 0,
    "boosterSeat": 0,
    "pickUpDescription": "Sydney International Airport (SYD)",
    "dropOffDescription": "Hilton Sydney",
    "legs": [
      {
        "id": "aaaaaaaa-1111-1111-1111-111111111111",
        "type": "arrival",
        "date": "2026-08-15",
        "time": "16:45",
        "flightNumber": "QF101",
        "channelFareType": "SYD-AR-Z1SED",
        "adults": 2,
        "children": 0,
        "pickUpDescription": "Sydney International Airport (SYD)",
        "dropOffDescription": "Hilton Sydney",
        "notes": "Flight delayed, new ETA 16:45"
      },
      {
        "id": "cccccccc-3333-3333-3333-333333333333",
        "type": "departure",
        "date": "2026-08-20",
        "time": "11:00",
        "flightNumber": "QF002",
        "channelFareType": "SYD-DE-Z1SED",
        "adults": 2,
        "children": 0,
        "pickUpDescription": "Hilton Sydney",
        "dropOffDescription": "Sydney International Airport (SYD)",
        "notes": "New departure car added"
      }
    ]
  },
  "accommodation_items": [
    {
      "id": "hotel-001",
      "reservation": {
        "check_in": "2026-08-15",
        "check_out": "2026-08-20",
        "hotel_info": {
          "id": "hilton-syd-001",
          "name": "Hilton Sydney",
          "geo_data": {
            "country": "Australia",
            "administrative_area_level_1": "New South Wales",
            "place_id": "ChIJrTLr-GyuEmsRBfy61i59si0"
          }
        }
      }
    }
  ]
}
```

**cURL:**
```bash
curl -X POST https://tourchain.icstravelgroup.com/tourchain/api/v2/IcsTransfer/webhook/booking-complete \
  -H "Authorization: Bearer <jwt_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "number": "TEST-001-NEW",
    "stopFollowUpButton": true,
    "customer_email": "newcustomer@example.com",
    "customer_given_name": "John",
    "customer_surname": "Smith",
    "customer_phone": "+61412345678",
    "transfer_information": {
      "checkInDate": "2026-08-15",
      "checkOutDate": "2026-08-20",
      "fullname": "John Smith",
      "countryCodePhone": "61",
      "phone": "412345678",
      "luggage": 2,
      "oversizeLuggage": 0,
      "babyCarSeat": 0,
      "boosterSeat": 0,
      "pickUpDescription": "Sydney International Airport (SYD)",
      "dropOffDescription": "Hilton Sydney",
      "legs": [
        {
          "id": "aaaaaaaa-1111-1111-1111-111111111111",
          "type": "arrival",
          "date": "2026-08-15",
          "time": "16:45",
          "flightNumber": "QF101",
          "channelFareType": "SYD-AR-Z1SED",
          "adults": 2,
          "children": 0,
          "pickUpDescription": "Sydney International Airport (SYD)",
          "dropOffDescription": "Hilton Sydney",
          "notes": "Flight delayed, new ETA 16:45"
        },
        {
          "id": "cccccccc-3333-3333-3333-333333333333",
          "type": "departure",
          "date": "2026-08-20",
          "time": "11:00",
          "flightNumber": "QF002",
          "channelFareType": "SYD-DE-Z1SED",
          "adults": 2,
          "children": 0,
          "pickUpDescription": "Hilton Sydney",
          "dropOffDescription": "Sydney International Airport (SYD)",
          "notes": "New departure car added"
        }
      ]
    },
    "accommodation_items": [
      {
        "id": "hotel-001",
        "reservation": {
          "check_in": "2026-08-15",
          "check_out": "2026-08-20",
          "hotel_info": {
            "id": "hilton-syd-001",
            "name": "Hilton Sydney",
            "geo_data": {
              "country": "Australia",
              "administrative_area_level_1": "New South Wales",
              "place_id": "ChIJrTLr-GyuEmsRBfy61i59si0"
            }
          }
        }
      }
    ]
  }'
```

**Expected Response:**
```json
{
  "status": 200,
  "message": "Booking accepted for processing"
}
```

**Expected State Change:** The existing arrival leg (`aaaaaaaa-1111-1111-1111-111111111111`) remains unchanged. The new departure leg (`cccccccc-3333-3333-3333-333333333333`) is created with `status: "confirmed"`. A new downstream service is created for it.

---

### Scenario 7: Missing Required Field

Attempt to create a booking without the required `number` field.

**Payload (missing `number`):**
```json
{
  "stopFollowUpButton": true,
  "customer_email": "newcustomer@example.com",
  "customer_given_name": "John",
  "customer_surname": "Smith",
  "customer_phone": "+61412345678",
  "transfer_information": {
    "fullname": "John Smith",
    "legs": [
      {
        "id": "test-id",
        "type": "arrival",
        "adults": 1
      }
    ]
  },
  "accommodation_items": []
}
```

**cURL:**
```bash
curl -X POST https://tourchain.icstravelgroup.com/tourchain/api/v2/IcsTransfer/webhook/booking-complete \
  -H "Authorization: Bearer <jwt_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "stopFollowUpButton": true,
    "customer_email": "newcustomer@example.com",
    "customer_given_name": "John",
    "customer_surname": "Smith",
    "customer_phone": "+61412345678",
    "transfer_information": {
      "fullname": "John Smith",
      "legs": [
        {
          "id": "test-id",
          "type": "arrival",
          "adults": 1
        }
      ]
    },
    "accommodation_items": []
  }'
```

**Expected Response (422):**
```json
{
  "status": 422,
  "error": 1,
  "messages": {
    "type": "error",
    "message": {
      "number": "Booking number is required"
    }
  }
}
```

---

### Scenario 8: Invalid Leg Data

Attempt to create a booking with duplicate leg `id`s.

**Payload (duplicate leg IDs):**
```json
{
  "number": "TEST-INVALID-001",
  "stopFollowUpButton": true,
  "customer_email": "test@example.com",
  "customer_given_name": "John",
  "customer_surname": "Smith",
  "customer_phone": "+61412345678",
  "transfer_information": {
    "fullname": "John Smith",
    "legs": [
      {
        "id": "duplicate-id",
        "type": "arrival",
        "adults": 2
      },
      {
        "id": "duplicate-id",
        "type": "departure",
        "adults": 2
      }
    ]
  },
  "accommodation_items": [
    {
      "id": "hotel-001",
      "reservation": {
        "check_in": "2026-08-15",
        "check_out": "2026-08-20",
        "hotel_info": {
          "id": "hilton-syd-001",
          "name": "Hilton Sydney"
        }
      }
    }
  ]
}
```

**cURL:**
```bash
curl -X POST https://tourchain.icstravelgroup.com/tourchain/api/v2/IcsTransfer/webhook/booking-complete \
  -H "Authorization: Bearer <jwt_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "number": "TEST-INVALID-001",
    "stopFollowUpButton": true,
    "customer_email": "test@example.com",
    "customer_given_name": "John",
    "customer_surname": "Smith",
    "customer_phone": "+61412345678",
    "transfer_information": {
      "fullname": "John Smith",
      "legs": [
        {
          "id": "duplicate-id",
          "type": "arrival",
          "adults": 2
        },
        {
          "id": "duplicate-id",
          "type": "departure",
          "adults": 2
        }
      ]
    },
    "accommodation_items": [
      {
        "id": "hotel-001",
        "reservation": {
          "check_in": "2026-08-15",
          "check_out": "2026-08-20",
          "hotel_info": {
            "id": "hilton-syd-001",
            "name": "Hilton Sydney"
          }
        }
      }
    ]
  }'
```

**Expected Response (422):**
```json
{
  "status": 422,
  "error": 1,
  "messages": {
    "type": "error",
    "message": {
      "transfer_information.legs": "Each leg id must be unique within legs"
    }
  }
}
```
